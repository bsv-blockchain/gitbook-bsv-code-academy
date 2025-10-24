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
- [Multi-Input Transaction](multi-input-transaction/README.md) - Combine multiple inputs
- [Change Output Calculation](change-output/README.md) - Calculate and create change
- [Fee Estimation](fee-estimation/README.md) - Estimate transaction fees

### Key Management
- [Generate Private Key](generate-private-key/README.md) - Create new private keys
- [Import/Export Keys](import-export-keys/README.md) - WIF and hex formats
- [HD Wallet Setup](hd-wallet-setup/README.md) - Create HD wallet structure
- [BRC-42 Key Derivation](brc-42-derivation/README.md) - Derive protocol-specific keys

### Signing & Verification
- [Sign Transaction](sign-transaction/README.md) - Sign with private keys
- [Verify Signature](verify-signature/README.md) - Verify digital signatures
- [Message Signing](message-signing/README.md) - Sign arbitrary messages
- [Multi-Sig Setup](multi-sig-setup/README.md) - Create multi-signature scripts

### Script Templates
- [P2PKH Template](p2pkh-template/README.md) - Standard payment script
- [Data Script](data-script/README.md) - OP_RETURN data storage
- [Custom Lock Script](custom-lock-script/README.md) - Build custom scripts
- [Script Evaluation](script-evaluation/README.md) - Test script execution

### Blockchain Verification
- [SPV Proof Verification](spv-verification/README.md) - Verify merkle proofs
- [BEEF Transaction](beef-transaction/README.md) - Create BEEF format
- [Block Header Validation](block-header-validation/README.md) - Validate headers

### Broadcasting
- [Broadcast with ARC](broadcast-arc/README.md) - Send transactions via ARC
- [Direct Broadcast](direct-broadcast/README.md) - Broadcast to nodes
- [Monitor Transaction Status](monitor-tx-status/README.md) - Track confirmations

### Data Operations
- [Encrypt Data](encrypt-data/README.md) - Encrypt with ECIES
- [Decrypt Data](decrypt-data/README.md) - Decrypt ECIES data
- [Store Data On-Chain](store-data-onchain/README.md) - Use OP_RETURN
- [Hash Operations](hash-operations/README.md) - SHA256, RIPEMD160, etc.

### Wallet Operations
- [Create Wallet](create-wallet/README.md) - Initialize new wallet
- [Restore Wallet](restore-wallet/README.md) - Restore from mnemonic
- [Check Balance](check-balance/README.md) - Query wallet balance
- [List UTXOs](list-utxos/README.md) - Get available UTXOs

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
