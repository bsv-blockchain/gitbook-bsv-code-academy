# BSV Code Academy

**Your Complete Guide to Building on Bitcoin SV**

Welcome to the BSV Code Academy - a comprehensive, modular learning platform for developers at all skill levels who want to build applications on the Bitcoin Satoshi Vision (BSV) blockchain. From beginner to intermediate production-ready applications.

---

## 📚 How to Use This Repository

This academy is organized into **three complementary systems** that work together:

### 1. 🎓 **Learning Paths** (Structured Courses)
**Start here for guided learning**

Follow progressive courses from beginner to intermediate. Each course teaches concepts and references the SDK Components and Code Features below.

- **When to use**: You want to learn systematically
- **Path**: Read courses in order, following the references to components
- **Time**: Full path ~40-60 hours

### 2. 🔧 **SDK Components** (Technical Reference)
**Use as reference documentation**

Deep-dive documentation for each part of the BSV SDK. Think of these as your technical manual - comprehensive API docs with examples.

- **When to use**: You need detailed technical specs or API reference
- **Path**: Jump directly to the component you need
- **Time**: 15-30 minutes per component

### 3. 💻 **Code Features** (Ready-to-Use Examples)
**Copy and adapt for your projects**

Production-ready code examples for specific features. Copy, modify, and use in your applications.

- **When to use**: You need working code for a specific task
- **Path**: Find the feature, copy the code, adapt to your needs
- **Time**: 5-15 minutes per feature

---

## 🚀 Quick Start Guides

### For Complete Beginners
**"I'm new to BSV blockchain development"**

```
1. Start → learning-paths/beginner/getting-started/
2. Follow → All 7 beginner courses in order
3. Practice → Use code-features examples
4. Reference → SDK components when needed
```

**Estimated time**: 16 hours
**You'll learn**: Development setup, BSV fundamentals, create wallet, build transactions, integrate wallets

### For Experienced Developers
**"I know blockchain, new to BSV"**

```
1. Quick scan → learning-paths/beginner/bsv-fundamentals/
2. Jump to → learning-paths/intermediate/
3. Reference → sdk-components/ for API details
4. Use → code-features/ for specific implementations
```

**Estimated time**: 8-12 hours to get productive
**You'll learn**: BSV-specific features, transaction building, SPV, BRC standards, payment systems, tokenization

### For SDK Reference
**"I need API documentation"**

```
1. Go to → sdk-components/
2. Find your component (e.g., transaction/, signatures/)
3. Read → API Reference section
4. Try → Common Patterns examples
```

**Estimated time**: 15-30 minutes per component
**You'll find**: Complete API docs, usage patterns, best practices

### For Code Examples
**"I need working code for [specific task]"**

```
1. Go to → code-features/
2. Find relevant feature (e.g., transaction-building/, multi-signature/)
3. Copy → Code examples
4. Adapt → To your use case
```

**Estimated time**: 5-15 minutes per feature
**You'll get**: Production-ready TypeScript code you can use immediately

---

## 📖 Learning Paths (Structured Courses)

### 🟢 Beginner Path
**Goal**: Learn BSV development fundamentals

| Course | Time | What You'll Learn |
|--------|------|-------------------|
| [Getting Started](learning-paths/beginner/getting-started/) | 2h | Install tools, understand workflow |
| [Development Environment](learning-paths/beginner/development-environment/) | 2h | Set up BSV development environment |
| [Development Paradigms](learning-paths/beginner/development-paradigms/) | 2h | BSV development patterns and paradigms |
| [BSV Fundamentals](learning-paths/beginner/bsv-fundamentals/) | 3h | Core concepts: UTXOs, transactions, scripts |
| [First Wallet](learning-paths/beginner/first-wallet/) | 2h | Create and manage BSV wallets |
| [First Transaction](learning-paths/beginner/first-transaction/) | 3h | Build, sign, and broadcast transactions |
| [Wallet Client Integration](learning-paths/beginner/wallet-client-integration/) | 2h | Integrate wallets into client applications |

**Total time**: ~16 hours
**Prerequisites**: Basic programming knowledge (JavaScript/TypeScript)
**Outcome**: Build simple BSV applications

### 🟡 Intermediate Path
**Goal**: Build production applications

#### Core Projects (Build Real Applications)

| Project | Time | What You'll Build |
|---------|------|-------------------|
| [Payment Systems](learning-paths/intermediate/payment-systems/) | 4-6h | Complete payment infrastructure with invoices, subscriptions, refunds |
| [Crowdfunding Platform](learning-paths/intermediate/crowdfunding-platform/) | 4-6h | Token-based crowdfunding with escrow and automated payouts |
| [Asset Tokenization](learning-paths/intermediate/asset-tokenization/) | 4-6h | Digital asset representation with compliance workflows |
| [Supply Chain Passports](learning-paths/intermediate/supply-chain-passports/) | 4-6h | Product lifecycle tracking with multi-party verification |

#### Supporting Modules (Reference as Needed)

| Module | Time | What You'll Learn |
|--------|------|-------------------|
| [BSV Primitives](learning-paths/intermediate/bsv-primitives/) | 2-3h | Deep dive into transaction structure, opcodes, UTXO management |
| [Transaction Building](learning-paths/intermediate/transaction-building/) | 3-4h | Multi-input transactions, batch payments, fee optimization |
| [Script Templates](learning-paths/intermediate/script-templates/) | 3-4h | Custom locking scripts, multisig, timelocks |
| [SPV Verification](learning-paths/intermediate/spv-verification/) | 3-4h | Merkle proofs, lightweight clients, BRC-9/67 |
| [BRC Standards](learning-paths/intermediate/brc-standards/) | 1.5-2h | BRC-29, BRC-42, protocol specifications |

**Total time**: 29-41 hours (16-24h projects + 13-17h modules)
**Prerequisites**: Complete beginner path
**Outcome**: Build four production-ready applications with full frontend and backend

---

## 🔧 SDK Components (Technical Reference)

**Complete API documentation for the BSV TypeScript SDK**

### Core Cryptography
- [Private Keys](sdk-components/private-keys/) - Key generation, WIF, signing
- [Public Keys](sdk-components/public-keys/) - Address derivation, verification
- [Signatures](sdk-components/signatures/) - ECDSA, DER encoding, SIGHASH types

### Transaction Components
- [Transaction](sdk-components/transaction/) - Build and manage transactions
- [Transaction Input](sdk-components/transaction-input/) - UTXO spending
- [Transaction Output](sdk-components/transaction-output/) - Locking scripts, OP_RETURN
- [Script](sdk-components/script/) - Bitcoin Script operations and opcodes
- [Script Templates](sdk-components/script-templates/) - Reusable script patterns

### Advanced Features
- [HD Wallets](sdk-components/hd-wallets/) - BRC-42/43 hierarchical derivation
- [UTXO Management](sdk-components/utxo-management/) - Coin selection strategies
- [SPV](sdk-components/spv/) - ChainTracker, header validation
- [Merkle Proofs](sdk-components/merkle-proofs/) - Proof creation and verification

### Network & Standards
- [P2PKH](sdk-components/p2pkh/) - Standard payment template
- [BEEF](sdk-components/beef/) - BRC-62 transaction envelopes
- [ARC](sdk-components/arc/) - Transaction broadcasting
- [BRC-29](sdk-components/brc-29/) - Payment protocol
- [BRC-42](sdk-components/brc-42/) - Key derivation standard

**How to use**: Click into any component for complete API reference, usage patterns, and examples.

---

## 💻 Code Features (Ready-to-Use Examples)

**Production-ready code you can copy and adapt**

### Transaction Operations
- [Transaction Building](code-features/transaction-building/) - Multi-input, batch payments
- [Transaction Creation](code-features/transaction-creation/) - Create transactions from scratch
- [Transaction Signing](code-features/transaction-signing/) - All SIGHASH types
- [Sign Transaction](code-features/sign-transaction/) - Transaction signing examples
- [Transaction Broadcasting](code-features/transaction-broadcasting/) - ARC integration
- [Broadcast ARC](code-features/broadcast-arc/) - ARC broadcasting examples
- [Transaction Chains](code-features/transaction-chains/) - BEEF format chains
- [Batch Operations](code-features/batch-operations/) - Efficient batch processing

### Wallet & Keys
- [Create Wallet](code-features/create-wallet/) - HD wallet creation
- [Generate Private Key](code-features/generate-private-key/) - Key generation
- [BRC-42 Derivation](code-features/brc-42-derivation/) - Hierarchical keys
- [UTXO Management](code-features/utxo-management/) - Selection strategies
- [Message Signing](code-features/message-signing/) - Sign and verify messages

### Scripts & Templates
- [P2PKH Template](code-features/p2pkh-template/) - Standard payments
- [Multi-Signature](code-features/multi-signature/) - M-of-N multisig
- [Custom Scripts](code-features/custom-scripts/) - Hash locks, timelocks
- [Custom Templates](code-features/custom-templates/) - Template creation
- [Standard Transactions](code-features/standard-transactions/) - P2PKH, P2PK

### Verification & Security
- [SPV](code-features/spv/) - SPV client implementation
- [SPV Verification](code-features/spv-verification/) - Merkle proof validation
- [SPV Validation](code-features/spv-validation/) - Transaction validation with SPV
- [Double Spend Detection](code-features/double-spend-detection/) - Security monitoring

### Advanced Patterns
- [Smart Contracts](code-features/smart-contracts/) - Escrow, vesting, oracles
- [Atomic Swaps](code-features/atomic-swaps/) - Trustless exchanges
- [Payment Channels](code-features/payment-channels/) - Off-chain payments
- [OP_RETURN](code-features/op-return/) - Data embedding

### Business Applications
- [Payment Processing](code-features/payment-processing/) - Invoices, subscriptions
- [Payment Distribution](code-features/payment-distribution/) - Split payments
- [E-commerce Integration](code-features/ecommerce-integration/) - Checkout systems
- [Marketplace](code-features/marketplace/) - Escrow patterns
- [Content Paywall](code-features/content-paywall/) - Paywalled content

**How to use**: Browse features, find what you need, copy the code, adapt to your project.

---

## 🎯 Learning by Use Case

### "I want to build a wallet"
```
1. Learning: beginner/first-wallet/ + intermediate/bsv-primitives/
2. Components: hd-wallets/, private-keys/, utxo-management/
3. Code: create-wallet/, utxo-management/, brc-42-derivation/
```

### "I want to create a token system"
```
1. Learning: intermediate/asset-tokenization/ + intermediate/script-templates/
2. Components: script-templates/, transaction/, spv/
3. Code: custom-scripts/, custom-templates/, transaction-building/
```

### "I want to integrate payments"
```
1. Learning: beginner/first-transaction/ + intermediate/transaction-building/
2. Components: transaction/, arc/, brc-29/
3. Code: payment-processing/, transaction-broadcasting/, batch-operations/
```

### "I want to build payment infrastructure"
```
1. Learning: intermediate/payment-systems/ + intermediate/transaction-building/
2. Components: transaction/, arc/, brc-29/
3. Code: payment-processing/, transaction-broadcasting/, batch-operations/
```

### "I want to build smart contracts"
```
1. Learning: intermediate/script-templates/ + intermediate/bsv-primitives/
2. Components: script/, script-templates/, signatures/
3. Code: smart-contracts/, custom-scripts/, multi-signature/
```

---

## 🗺️ Complete Repository Map

```
gitbook-bsv-code-academy/
│
├── learning-paths/          # 📚 Structured courses (follow in order)
│   ├── beginner/           # 7 courses (~16 hours)
│   └── intermediate/       # 9 courses: 4 projects + 5 modules (~29-41 hours)
│
├── sdk-components/         # 🔧 Technical reference (jump to as needed)
│   ├── private-keys/       # API docs + patterns
│   ├── transaction/        # API docs + patterns
│   ├── script/            # API docs + patterns
│   └── ... (17 components total)
│
├── code-features/          # 💻 Ready-to-use code (copy & adapt)
│   ├── transaction-building/    # Production examples
│   ├── multi-signature/         # Production examples
│   ├── payment-processing/      # Production examples
│   └── ... (31 features total)
│
├── assets/                 # 🎨 Images, diagrams, icons
└── resources/              # 📚 External links and references
```

---

## 📊 What's Inside

- **📚 16 Structured Courses** covering beginner → intermediate
  - 7 Beginner courses
  - 9 Intermediate courses (4 projects + 5 modules)
- **🔧 17 SDK Components** with complete API documentation
- **💻 31 Code Features** with production-ready examples
- **📝 Comprehensive documentation** covering all aspects of BSV development
- **⚡ Latest Tech** including Teranode architecture and deployment

---

## 🛠️ Installation & Setup

### Prerequisites
- Node.js 18+
- TypeScript 4.9+
- Basic terminal/command line knowledge

### Install BSV SDK
```bash
npm install @bsv/sdk
```

### Quick Test
```typescript
import { PrivateKey } from '@bsv/sdk'

const key = PrivateKey.fromRandom()
console.log('Address:', key.toPublicKey().toAddress())
```

**Full setup guide**: [learning-paths/beginner/development-environment/](learning-paths/beginner/development-environment/)

---

## 🤝 How Components Work Together

### Modular Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    LEARNING PATHS                       │
│  (Teach concepts, guide through building applications)  │
│                          ↓                              │
│                    References ↓                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   SDK COMPONENTS                        │
│     (Technical reference - "What is this? How does      │
│      this API work? What are the methods?")             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   CODE FEATURES                         │
│   (Production examples - "Show me working code!")       │
└─────────────────────────────────────────────────────────┘
```

**Example Flow**:
1. You're learning about transactions → Read `learning-paths/intermediate/transaction-building/`
2. Need API details → Reference `sdk-components/transaction/`
3. Want working code → Copy from `code-features/transaction-building/`

---

## 🎓 Recommended Learning Sequences

### Fastest Path to Production (2 weeks intensive)
```
Week 1: Beginner essentials (getting-started, fundamentals, first-wallet, first-transaction)
Week 2: Intermediate skills (transaction-building, script-templates) + Your use case
       + code-features implementation
```

### Comprehensive Mastery (2-3 months part-time)
```
Month 1: All beginner courses (7 courses, ~16 hours)
Month 2: Intermediate projects (payment-systems, crowdfunding-platform)
        + supporting modules as needed (BSV primitives, transaction-building)
Month 3: More intermediate projects (asset-tokenization, supply-chain-passports)
        + remaining modules (script-templates, SPV, BRC standards)
```

### SDK Reference User (ongoing)
```
- Keep sdk-components/ bookmarked
- Jump to specific components as needed
- Use code-features/ for implementation patterns
```

---

## 🌟 Key Features

✅ **Complete Coverage** - From "Hello World" to running Teranode infrastructure
✅ **Production Ready** - All code examples tested with real @bsv/sdk API
✅ **Modular Design** - Use what you need, skip what you don't
✅ **Latest Technology** - Includes Teranode, latest BRC standards
✅ **TypeScript First** - Modern, type-safe development
✅ **Cross-Referenced** - Easy navigation between related topics

---

## 📚 External Resources

### Official BSV Resources
- [BSV Blockchain](https://www.bsvblockchain.org/) - Official BSV website
- [BSV Skills Center](https://hub.bsvblockchain.org/bsv-skills-center) - Official learning hub
- [Teranode Docs](https://bsv-blockchain.github.io/teranode/) - Teranode documentation
- [BRC Standards Hub](https://hub.bsvblockchain.org/brc) - BSV protocol standards
- [BRC GitHub Repository](https://github.com/bitcoin-sv/BRCs) - BRC specifications source

### Developer Tools
- [BSV TypeScript SDK](https://github.com/bsv-blockchain/ts-sdk) - Official SDK repository
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk/) - SDK API reference
- [MetaNet Desktop Wallet](https://desktop.bsvb.tech/) - BSV desktop wallet
- [Wallet Toolbox](https://fast.brc.dev/) - WalletClient API documentation
- [Get BSV - Orange Gateway](https://hub.bsvblockchain.org/demos-and-onboardings/onboardings/onboarding-catalog/get-bsv/orange-gateway) - Purchase BSV with fiat
- [WhatsOnChain](https://whatsonchain.com/) - BSV blockchain explorer
- [Gorilla Pool](https://gorillapool.io/) - Mining pool and tools

### Community
- [BSV Discord](https://discord.gg/bsv) - Developer community
- [BSV Developers](https://www.reddit.com/r/bsvdevs/) - Reddit community
- [Stack Overflow](https://stackoverflow.com/questions/tagged/bsv) - Q&A (tag: bsv)

---

## 🤝 Contributing

This is a living documentation project. Contributions are welcome!

- Found an error? [Open an issue](https://github.com/bsv-blockchain/gitbook-bsv-code-academy/issues)
- Have improvements? Submit a pull request
- Want to add examples? We'd love to see them!

---

## 📄 License

This documentation is provided as-is for educational purposes.

---

## 🚀 Get Started Now!

**Choose your path**:

- 🆕 **New to BSV?** → [Start Learning](learning-paths/beginner/getting-started/)
- 📖 **Need API docs?** → [Browse SDK Components](sdk-components/)
- 💻 **Want code examples?** → [Explore Code Features](code-features/)

**Happy Building on BSV! 🎉**

---

*Built with ❤️ for the BSV developer community*
