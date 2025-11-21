# Getting Started with BSV

## Introduction

Welcome to the BSV Code Academy! This guide will introduce you to the Bitcoin Satoshi Vision (BSV) blockchain and help you understand what makes it unique and powerful for application development.

## What is BSV Blockchain?

Bitcoin Satoshi Vision (BSV) is a blockchain that restores the original Bitcoin protocol as described in Satoshi Nakamoto's whitepaper. BSV focuses on:

- **Massive Scalability**: Unbounded block sizes for global-scale applications
- **Stable Protocol**: Set-in-stone protocol for reliable long-term development
- **Low Fees**: Micropayment-friendly transaction costs
- **Rich Data**: Store and manage data directly on-chain
- **UTXO Model**: Efficient parallel transaction processing

## Why Build on BSV?

### For Application Developers

1. **True Data Ownership**: Store immutable data on-chain
2. **Micropayments**: Enable new business models with sub-cent transactions
3. **Identity & Authentication**: Built-in cryptographic identity
4. **No Smart Contract Limitations**: Turing complete scripting with no gas limits
5. **Instant Payment Verification**: SPV for fast, secure confirmations

### Use Cases

- **Digital Identity**: Self-sovereign identity solutions
- **Supply Chain**: Immutable tracking and verification
- **Social Networks**: Censorship-resistant platforms
- **Tokenization**: Asset representation and transfer
- **Data Marketplace**: Buy and sell data with micropayments
- **IoT**: Machine-to-machine payments
- **Gaming**: True item ownership and trading

## Key Differences from Other Blockchains

### vs Bitcoin (BTC)
- **Block Size**: BSV has unbounded blocks; BTC is limited to 1-4MB
- **Focus**: BSV prioritizes scaling and data; BTC focuses on store of value
- **Fees**: BSV has consistently low fees; BTC fees vary widely
- **Protocol**: BSV has stable protocol; BTC frequently changes

### vs Ethereum
- **Architecture**: BSV uses UTXO model; Ethereum uses account model
- **Fees**: BSV has predictable fees; Ethereum gas fees fluctuate
- **Scaling**: BSV scales on-chain; Ethereum uses Layer 2
- **Scripting**: BSV uses Bitcoin Script; Ethereum uses Solidity

### vs Other Layer 1s
- **Proven Security**: BSV uses proven proof-of-work
- **Data Storage**: BSV supports large data on-chain
- **Stability**: BSV protocol is stable; others frequently update

## Core Concepts

### 1. UTXO Model
Transactions consume unspent transaction outputs (UTXOs) and create new ones. This enables:
- Parallel processing
- Clear ownership
- Efficient verification

### 2. Satoshis
The base unit of BSV:
- 1 BSV = 100,000,000 satoshis
- Enables micropayments
- All amounts are integers (no floating point)

### 3. Keys and Addresses
- **Private Key**: Secret key for signing (keep secret!)
- **Public Key**: Derived from private key (can share)
- **Address**: Hash of public key (for receiving payments)

### 4. Transactions
- **Inputs**: Reference previous outputs being spent
- **Outputs**: Create new spendable outputs
- **Scripts**: Lock/unlock conditions

### 5. Blocks and Confirmations
- **Block**: Bundle of transactions
- **Confirmation**: Transaction included in a block
- **More Confirmations**: More security (6+ is standard)

## BSV Ecosystem Overview

### Network Components

1. **Nodes**: Miners who process transactions and create blocks
2. **Wallets**: Applications for managing keys and transactions
3. **ARC**: Simplified transaction broadcasting
4. **Overlays**: Application-specific network layers
5. **SPV Servers**: Provide merkle proofs for verification

### Developer Tools

1. **BSV TypeScript SDK**: Primary development toolkit
2. **BSV Libraries**: Go, Python, and other language SDKs
3. **Block Explorers**: View blockchain data (WhatsOnChain)
4. **Testnet**: Safe testing environment
5. **ARC API**: Transaction broadcasting service

### Standards (BRC)

Bitcoin Request for Comments (BRC) define interoperability standards:
- **BRC-42**: Key derivation protocol
- **BRC-43**: Wallet security levels
- **BRC-29**: Payment protocol
- **BRC-56**: Certificate management
- And many more...

## The BSV Code Academy Structure

This academy is organized into three learning paths:

### Beginner (You are here!)
- Setting up your environment
- Understanding fundamentals
- Creating wallets and transactions
- Basic blockchain interactions

### Intermediate
- Advanced transaction types
- BSV primitives and protocols
- BRC standards implementation
- SPV verification

### Advanced
- Network topology
- Node operations
- Custom overlay networks
- Complex smart contracts

## What You'll Build

Throughout this beginner path, you'll build:

1. **A Basic Wallet**: Generate keys and addresses
2. **A Payment Application**: Send and receive BSV
3. **Transaction Monitor**: Track confirmations

These projects will give you hands-on experience with BSV development.

## Development Philosophy

BSV development follows these principles:

1. **Protocol Stability**: Write code that works long-term
2. **On-Chain First**: Leverage the blockchain's capabilities
3. **Micropayment Native**: Design for micro-transactions
4. **Data Ownership**: Put users in control of their data
5. **SPV Security**: Trust through verification, not third parties

## Next Steps

Now that you understand what BSV is and why it's powerful, let's set up your development environment!

Continue to: [Development Environment](../development-environment/README.md)

## Additional Resources

- [BSV Whitepaper](https://bitcoinsv.io/bitcoin.pdf) - Original Bitcoin whitepaper
- [BSV Wiki](https://wiki.bitcoinsv.io/) - Comprehensive BSV information
- [BSV Skills Center](https://hub.bsvblockchain.org/bsv-skills-center) - Official learning resources
- [WhatsOnChain](https://whatsonchain.com/) - BSV block explorer

## Questions to Consider

Before moving forward, make sure you understand:
- What makes BSV different from other blockchains?
- What is the UTXO model?
- What are satoshis?
- Why is protocol stability important?

Ready? Let's set up your development environment!
