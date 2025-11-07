# Code Features

This directory contains reusable code snippets and complete examples demonstrating specific BSV blockchain functionality. These code features are used as building blocks throughout the learning paths.

## Purpose

Code Features provide:
- **Practical Examples**: Working code that demonstrates real-world usage
- **Reusable Snippets**: Copy-paste ready code for common tasks
- **Best Practices**: Implementation patterns that follow BSV conventions
- **Modular Components**: Code that can be combined for complex solutions

## Organization

Features are organized by functionality and use case. Each feature includes:
- Complete, runnable code examples
- Explanation of what the code does
- Prerequisites and dependencies
- Common variations and extensions
- Links to related SDK components

## Available Code Features

### Transaction Operations
- [Transaction Creation](transaction-creation/README.md) - Create basic transactions
- [Transaction Building](transaction-building/README.md) - Build complex transactions
- [Transaction Signing](transaction-signing/README.md) - Sign transactions
- [Transaction Broadcasting](transaction-broadcasting/README.md) - Broadcast transactions
- [Transaction Chains](transaction-chains/README.md) - Create transaction chains
- [Standard Transactions](standard-transactions/README.md) - Common transaction patterns
- [Batch Operations](batch-operations/README.md) - Process multiple transactions
- [UTXO Management](utxo-management/README.md) - Manage unspent outputs

### Key Management & Signing
- [Generate Private Key](generate-private-key/README.md) - Create new private keys
- [BRC-42 Key Derivation](brc-42-derivation/README.md) - Derive protocol-specific keys
- [Sign Transaction](sign-transaction/README.md) - Sign with private keys
- [Message Signing](message-signing/README.md) - Sign arbitrary messages
- [Multi-Signature](multi-signature/README.md) - Multi-signature transactions

### Script Templates
- [P2PKH Template](p2pkh-template/README.md) - Standard payment script
- [Custom Scripts](custom-scripts/README.md) - Build custom locking scripts
- [Custom Templates](custom-templates/README.md) - Custom script templates
- [OP_RETURN](op-return/README.md) - Data storage on-chain

### Blockchain Verification
- [SPV](spv/README.md) - Simplified Payment Verification
- [SPV Verification](spv-verification/README.md) - Verify merkle proofs
- [SPV Validation](spv-validation/README.md) - Validate SPV proofs

### Broadcasting
- [Broadcast with ARC](broadcast-arc/README.md) - Send transactions via ARC

### Wallet Operations
- [Create Wallet](create-wallet/README.md) - Initialize new wallet

### Advanced Features
- [Payment Channels](payment-channels/README.md) - Bi-directional payment channels
- [Payment Distribution](payment-distribution/README.md) - Distribute payments
- [Payment Processing](payment-processing/README.md) - Process payments
- [Atomic Swaps](atomic-swaps/README.md) - Trustless token exchanges
- [Smart Contracts](smart-contracts/README.md) - Smart contract patterns
- [Double-Spend Detection](double-spend-detection/README.md) - Detect double-spends
- [Content Paywall](content-paywall/README.md) - Micropayment paywalls
- [E-commerce Integration](ecommerce-integration/README.md) - Online store integration
- [Marketplace](marketplace/README.md) - Peer-to-peer marketplace

## How to Use

Each code feature directory contains:
1. `README.md` - Feature overview and explanation
2. `example.ts` - Complete TypeScript example
3. `variations.md` - Alternative approaches (if applicable)

### In Learning Paths
Features are referenced in learning modules:
```markdown
See [Transaction Creation](../../code-features/transaction-creation/README.md) for a complete example.
```

### As Templates
Copy the example code and adapt to your needs:
```typescript
// From code-features/transaction-creation/example.ts
import { Transaction } from '@bsv/sdk'
// ... rest of the code
```

## Code Style

All examples follow these conventions:
- **TypeScript**: Fully typed for clarity
- **Async/Await**: Modern async patterns
- **Error Handling**: Proper try/catch blocks
- **Comments**: Explain key concepts
- **Imports**: Show all required dependencies

## Testing Examples

Most features include test cases showing expected behavior:
```typescript
// example.test.ts
describe('Transaction Creation', () => {
  it('should create a valid transaction', () => {
    // test code
  })
})
```

## Dependencies

Code features assume you have installed:
```json
{
  "@bsv/sdk": "^1.0.0"
}
```

Additional dependencies are noted in each feature's README.

## Contributing

When adding new code features:
1. Create a descriptive folder name (kebab-case)
2. Include working, tested code
3. Add comprehensive comments
4. Document prerequisites
5. Link related SDK components
6. Provide variations when useful

## Feature Status

- ✅ Essential features documented
- 🚧 Additional features in progress
- 📋 Planned features

## Related Documentation

- [SDK Components](../sdk-components/README.md) - API reference for components used in these examples
- [Learning Paths](../learning-paths/) - Guided tutorials using these features
