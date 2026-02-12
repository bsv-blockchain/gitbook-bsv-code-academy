# BSV Code Academy

**Your Complete Guide to Building on BSV Blockchain**

Welcome to the BSV Code Academy - a comprehensive, modular learning platform for developers at all skill levels who want to build applications on the Bitcoin Satoshi Vision (BSV) blockchain. This academy includes **20 structured courses** (7 beginner + 2 intermediate concepts + 5 intermediate projects + 5 advanced modules + 1 advanced project), **17 SDK components**, and **31 code features** covering ~39-49 hours of content from fundamentals to production-ready applications.

---

## 📚 How to Use This Repository

This academy is organized into **three complementary systems** that work together:

### 1. 🎓 **Learning Paths** (Structured Courses)
**Start here for guided learning**

Follow progressive courses from beginner to advanced. Each course teaches concepts and references the SDK Components and Code Features below.

- **When to use**: You want to learn systematically
- **Path**: Read courses in order, following the references to components
- **Time**: Full path ~39-49 hours

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

**Estimated time**: ~5 hours
**You'll learn**: Development setup, BSV fundamentals, create wallet, build transactions, integrate wallets

### For Experienced Developers
**"I know blockchain, new to BSV"**

```
1. Quick scan → learning-paths/beginner/bsv-fundamentals/
2. Jump to → learning-paths/intermediate/
3. Reference → sdk-components/ for API details
4. Use → code-features/ for specific implementations
```

**Estimated time**: 3-4 hours to get productive
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
| [Getting Started](learning-paths/beginner/getting-started/) | 30 min | Install tools, understand workflow |
| [Development Environment](learning-paths/beginner/development-environment/) | 45 min | Set up BSV development environment |
| [Development Paradigms](learning-paths/beginner/development-paradigms/) | 30 min | BSV development patterns and paradigms |
| [BSV Fundamentals](learning-paths/beginner/bsv-fundamentals/) | 1h | Core concepts: UTXOs, transactions, scripts |
| [First Wallet](learning-paths/beginner/first-wallet/) | 45 min | Create and manage BSV wallets |
| [First Transaction](learning-paths/beginner/first-transaction/) | 1h | Build, sign, and broadcast transactions |
| [Wallet Client Integration](learning-paths/beginner/wallet-client-integration/) | 45 min | Integrate wallets into client applications |

**Total time**: ~5 hours
**Prerequisites**: Basic programming knowledge (JavaScript/TypeScript)
**Outcome**: Build simple BSV applications

### 🟡 Intermediate Path
**Goal**: Build production applications

#### Core Concepts (Learn Key Technologies)

| Course | Time | What You'll Learn |
|--------|------|-------------------|
| [Token Fundamentals](learning-paths/intermediate/token-fundamentals/) | 45-60m | PushDrop tokens, two-phase signing, MessageBox transfers |
| [Overlay Services](learning-paths/intermediate/overlay-services/) | 45-60m | Topic Managers, Lookup Services, protocol validation |

#### Hands-On Projects (Build Real Applications)

| Project | Time | What You'll Build |
|---------|------|-------------------|
| [Desktop Wallet Management](learning-paths/intermediate/desktop-wallet-management/) | 2-3h | Wallet data integration, key derivation, baskets |
| [Inscription Platform](learning-paths/intermediate/inscription-platform/) | 2-3h | On-chain data storage with OP_RETURN, basket organization, file hashing |
| [MessageBox Platform](learning-paths/intermediate/messagebox-platform/) | 3-4h | P2P encrypted messaging, identity certification, BRC-29 payments |
| [Crowdfunding Platform](learning-paths/intermediate/crowdfunding-platform/) | 4-6h | Token-based crowdfunding with escrow and automated payouts |
| [Utilitary Token Platform](learning-paths/intermediate/utilitary-token-platform/) | 6-8h | Full-stack token system with overlay validation and MongoDB indexing |

**Total time**: 19-26 hours
**Prerequisites**: Complete beginner path
**Outcome**: Build production-ready applications with full frontend and backend

### 🔴 Advanced Path
**Goal**: Build enterprise-grade systems with authentication and custom protocols

#### Hands-On Projects

| Project | Time | What You'll Build |
|---------|------|-------------------|
| [Certification Platform](learning-paths/advanced/certification-platform/) | 4-6h | Self-hosted certificate issuance, field encryption, mutual auth, gated access |

#### Learning Modules

| Module | Time | What You'll Learn |
|--------|------|-------------------|
| [Overlay Networks](learning-paths/advanced/overlay-networks/) | 3h | Application-specific networks, UTXO state management |
| [Network Topology](learning-paths/advanced/network-topology/) | 2.5h | P2P protocols, peer discovery, network security |
| [Node Operations](learning-paths/advanced/node-operations/) | 3h | Node setup, configuration, monitoring |
| [Advanced Scripting](learning-paths/advanced/advanced-scripting/) | 2.5h | Complex Bitcoin Script, oracle integration |
| [Custom Protocols](learning-paths/advanced/custom-protocols/) | 3h | Protocol design, versioning, interoperability |

**Total time**: 18-22 hours
**Prerequisites**: Complete intermediate path
**Outcome**: Build enterprise certification systems, overlay networks, and custom protocols

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

<!-- ### "I want to create a token system"
```
1. Learning: intermediate/asset-tokenization/ + intermediate/script-templates/
2. Components: script-templates/, transaction/, spv/
3. Code: custom-scripts/, custom-templates/, transaction-building/
``` -->

### "I want to integrate payments"
```
1. Learning: beginner/first-transaction/
2. Components: transaction/, arc/, brc-29/
3. Code: payment-processing/, transaction-broadcasting/, batch-operations/
```

<!-- ### "I want to build payment infrastructure"
```
1. Learning: intermediate/payment-systems/ + intermediate/transaction-building/
2. Components: transaction/, arc/, brc-29/
3. Code: payment-processing/, transaction-broadcasting/, batch-operations/
``` -->

<!-- ### "I want to build smart contracts"
```
1. Learning: intermediate/script-templates/ + intermediate/bsv-primitives/
2. Components: script/, script-templates/, signatures/
3. Code: smart-contracts/, custom-scripts/, multi-signature/
``` -->

---

## 🗺️ Complete Repository Map

```
gitbook-bsv-code-academy/
│
├── learning-paths/          # 📚 Structured courses (follow in order)
│   ├── beginner/           # 7 courses (~5 hours)
│   ├── intermediate/       # 2 courses + 5 projects (~19-26 hours)
│   └── advanced/           # 5 modules + 1 project (~18-22 hours)
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

- **📚 20 Structured Courses** covering beginner → advanced
  - 7 Beginner courses
  - 2 Intermediate concept courses
  - 5 Intermediate hands-on projects
  - 5 Advanced learning modules
  - 1 Advanced hands-on project (Certification Platform)
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
1. You're learning about transactions → Read `learning-paths/beginner/first-transaction/`
2. Need API details → Reference `sdk-components/transaction/`
3. Want working code → Copy from `code-features/transaction-building/`

---

## 🎓 Recommended Learning Sequences

### Fastest Path to Production (2 weeks intensive)
```
Week 1: Beginner essentials (7 courses, ~5 hours)
Week 2: Intermediate concepts (token-fundamentals, overlay-services, ~2 hours)
        + Start hands-on projects (desktop-wallet-management or inscription-platform)
```

### Comprehensive Mastery (6-7 weeks part-time)
```
Week 1: All beginner courses (7 courses, ~5 hours)
Week 2: Intermediate concepts (token-fundamentals, overlay-services, ~2 hours)
        + Desktop Wallet Management project (2-3 hours)
Week 3: Inscription Platform project (2-3 hours)
        + MessageBox Platform project (3-4 hours)
Week 4: Crowdfunding Platform project (4-6 hours)
Week 5: Utilitary Token Platform project (6-8 hours)
Week 6: Certification Platform project (4-6 hours)
        + Advanced modules (overlay networks, scripting)
Week 7: Remaining advanced modules + advanced projects
```

### Token-Focused Path
```
Week 1: Beginner essentials (focus on transactions and wallets)
Week 2: Token Fundamentals + Overlay Services concepts
Week 3-4: Utilitary Token Platform project (full-stack token system)
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
