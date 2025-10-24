# Quick Start Guide

Welcome to BSV Code Academy! This guide will help you get started quickly.

## For New Learners

### 1. Start with the Beginner Path
Begin your journey here: [Beginner Learning Path](learning-paths/beginner/README.md)

**First Steps**:
1. [Getting Started](learning-paths/beginner/getting-started/README.md) - Understand what BSV is
2. [Development Environment](learning-paths/beginner/development-environment/README.md) - Set up your tools
3. [BSV Fundamentals](learning-paths/beginner/bsv-fundamentals/README.md) - Learn core concepts
4. [Your First Wallet](learning-paths/beginner/first-wallet/README.md) - Create a wallet
5. [Your First Transaction](learning-paths/beginner/first-transaction/README.md) - Send BSV

**Time Required**: 4-5 hours

### 2. Progress to Intermediate
Ready to build real apps? [Intermediate Learning Path](learning-paths/intermediate/README.md)

**Key Topics**:
- Complex transaction building
- Bitcoin Script and templates
- SPV verification
- BRC standards (42, 43, 29, 56)

**Time Required**: 9-10 hours

### 3. Master Advanced Topics
Building infrastructure? [Advanced Learning Path](learning-paths/advanced/README.md)

**Focus Areas**:
- Overlay networks
- Node operations
- Custom protocols
- Enterprise patterns

**Time Required**: 14-16 hours

## For Experienced Developers

### Quick Reference

**Need API Documentation?**
→ Browse [SDK Components](sdk-components/README.md)

**Need Code Examples?**
→ Check [Code Features](code-features/README.md)

**Looking for Specific Topics?**
- Transaction building → [Transaction Component](sdk-components/transaction/README.md)
- Key management → [BRC-42](sdk-components/brc-42/README.md)
- Broadcasting → [ARC Component](sdk-components/arc/README.md)
- Verification → [SPV Component](sdk-components/spv/README.md)

## Common Use Cases

### "I want to build a payment app"
1. [Generate Private Key](code-features/generate-private-key/README.md)
2. [Create Wallet](code-features/create-wallet/README.md)
3. [Transaction Creation](code-features/transaction-creation/README.md)
4. [Broadcast with ARC](code-features/broadcast-arc/README.md)

### "I need to implement BRC-42 key derivation"
1. [BRC-42 Component](sdk-components/brc-42/README.md) - Understanding
2. [BRC-42 Derivation](code-features/brc-42-derivation/README.md) - Implementation
3. [BRC Standards](learning-paths/intermediate/brc-standards/README.md) - Full guide

### "I'm building an overlay network"
1. [Overlay Networks Guide](learning-paths/advanced/overlay-networks/README.md)
2. [SPV Verification](code-features/spv-verification/README.md)
3. [Transaction Indexing](learning-paths/advanced/overlay-networks/README.md#indexing-strategy)

### "I need to verify transactions with SPV"
1. [SPV Component](sdk-components/spv/README.md)
2. [Merkle Proofs](sdk-components/merkle-proofs/README.md)
3. [SPV Verification Example](code-features/spv-verification/README.md)

## Installation

```bash
# Create new project
mkdir my-bsv-app
cd my-bsv-app

# Initialize npm
npm init -y

# Install BSV SDK
npm install @bsv/sdk

# Install TypeScript
npm install -D typescript @types/node ts-node

# Create TypeScript config
npx tsc --init
```

## Your First BSV Code

```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate a private key
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()
const address = publicKey.toAddress()

console.log('Address:', address)
```

## Project Structure

```
gitbook-bsv-code-academy/
├── learning-paths/        # Guided tutorials
│   ├── beginner/         # Start here
│   ├── intermediate/     # Build apps
│   └── advanced/         # Infrastructure
├── sdk-components/        # API reference
├── code-features/         # Code examples
├── assets/               # Images & diagrams
└── resources/            # External links
```

## Navigation Tips

### By Skill Level
- **Beginner** → [Beginner Path](learning-paths/beginner/README.md)
- **Intermediate** → [Intermediate Path](learning-paths/intermediate/README.md)
- **Advanced** → [Advanced Path](learning-paths/advanced/README.md)

### By Topic
- **Transactions** → [Transaction Component](sdk-components/transaction/README.md)
- **Keys & Wallets** → [Private Keys](sdk-components/private-keys/README.md)
- **Scripts** → [Script Templates](learning-paths/intermediate/script-templates/README.md)
- **Standards** → [BRC Standards](learning-paths/intermediate/brc-standards/README.md)
- **Overlays** → [Overlay Networks](learning-paths/advanced/overlay-networks/README.md)

### By Use Case
- **Payment Apps** → [Transaction Creation](code-features/transaction-creation/README.md)
- **Identity** → [BRC-42](sdk-components/brc-42/README.md)
- **Data Storage** → [Overlay Networks](learning-paths/advanced/overlay-networks/README.md)
- **Verification** → [SPV](sdk-components/spv/README.md)

## Getting Help

### Documentation
- Check the relevant learning path module
- Browse SDK component docs
- Review code feature examples

### External Resources
- [BSV Skills Center](https://hub.bsvblockchain.org/bsv-skills-center)
- [BSV Wiki](https://wiki.bitcoinsv.io/)
- [WhatsOnChain](https://whatsonchain.com/) - Block explorer

### Community
- [External Resources](resources/external-links.md) - Full list of resources

## Next Steps

1. **Read the Overview** → [Main README](README.md)
2. **Choose Your Path** → Pick beginner/intermediate/advanced
3. **Start Building** → Use code features as templates
4. **Reference Docs** → SDK components for details

## Tips for Success

✅ **Follow the Path**: Complete modules sequentially
✅ **Write Code**: Type out examples, don't just read
✅ **Use Testnet**: Practice with free testnet BSV
✅ **Ask Questions**: Engage with the community
✅ **Build Projects**: Apply what you learn

---

Ready to start? Go to [Getting Started](learning-paths/beginner/getting-started/README.md)!
