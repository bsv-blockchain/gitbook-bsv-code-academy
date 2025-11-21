# Transaction Component

## Overview

The Transaction component is the fundamental building block for creating and managing BSV blockchain transactions. It provides a comprehensive API for constructing, signing, and serializing transactions.

## Purpose

Transactions are the core mechanism for transferring value and storing data on the BSV blockchain. This component handles:
- Creating new transactions
- Adding inputs and outputs
- Fee calculation
- Transaction signing
- Serialization for broadcasting

## Basic Usage

```typescript
import { Transaction } from '@bsv/sdk'

// Create a new transaction
const tx = new Transaction()

// Add inputs and outputs
tx.addInput(/* ... */)
tx.addOutput(/* ... */)

// Sign the transaction
await tx.sign()

// Serialize for broadcasting
const rawTx = tx.toHex()
```

## Key Features

### Transaction Creation
- Build transactions from scratch
- Parse existing transactions
- Clone and modify transactions

### Input/Output Management
- Add multiple inputs
- Create various output types
- Calculate change outputs

### Fee Management
- Automatic fee calculation
- Custom fee rates
- Fee estimation

### Signing
- Sign with private keys
- Multi-signature support
- Custom signing algorithms

## Related Components

- [Transaction Input](../transaction-input/README.md) - Managing transaction inputs
- [Transaction Output](../transaction-output/README.md) - Creating outputs
- [UTXO Management](../utxo-management/README.md) - Working with UTXOs
- [Signatures](../signatures/README.md) - Digital signature operations

## Code Examples

See [Code Features - Transaction Creation](../../code-features/transaction-creation/README.md) for complete examples.

## API Reference

### Constructor
```typescript
new Transaction(version?: number, inputs?: TransactionInput[], outputs?: TransactionOutput[], lockTime?: number)
```

### Methods
- `addInput(input: TransactionInput): void`
- `addOutput(output: TransactionOutput): void`
- `sign(): Promise<void>`
- `toHex(): string`
- `toBuffer(): Buffer`
- `getFee(): number`

## Common Patterns

### Simple Payment Transaction
Create a basic P2PKH payment transaction with automatic fee calculation.

### Data Storage Transaction
Include data in OP_RETURN outputs for on-chain storage.

### Multi-Input Transaction
Combine multiple UTXOs efficiently.

## Best Practices

1. Always verify inputs before signing
2. Use appropriate fee rates for timely confirmation
3. Validate output amounts
4. Handle errors during signing
5. Keep transactions under size limits

## Learning Path References

- Beginner: [Your First Transaction](../../learning-paths/beginner/first-transaction/README.md)
- Intermediate: [Transaction Building](../../learning-paths/intermediate/transaction-building/README.md)
