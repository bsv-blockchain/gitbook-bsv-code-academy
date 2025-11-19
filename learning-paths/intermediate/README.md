# Intermediate Learning Path

Welcome to the Intermediate Learning Path! This path takes you beyond the fundamentals to build **real-world, production-ready BSV blockchain applications** through hands-on project development.

## What Makes This Path Different

The intermediate path is **project-based**. Instead of learning concepts in isolation, you'll build three complete, deployable applications that demonstrate real blockchain use cases. Each project includes:

- **Full working code** you can run and deploy
- **Backend implementation** (custodial wallet management)
- **Frontend implementation** (user-controlled wallets with WalletClient + MetaNet Desktop Wallet)
- **Best practices** for production deployment
- **SDK-first approach** using pre-built components

## What You'll Build

By completing this path, you will build one production-ready application:

<!-- 0. **Payment Systems** - Complete payment infrastructure with invoicing, subscriptions, and refunds -->
1. **Crowdfunding Platform** - Token-based campaign funding with escrow and automated payouts
<!-- 2. **Asset Tokenization System** - Real-world asset representation with compliance workflows -->
<!-- 3. **Supply Chain Digital Passports** - Product lifecycle tracking with multi-party verification -->

## Prerequisites

Before starting this path, you should have completed the [Beginner Learning Path](../beginner/README.md) or have equivalent knowledge:
- BSV SDK installation and setup
- Creating and signing basic transactions
- Understanding UTXOs and transaction structure
- Wallet management with both backend and frontend approaches
- Using WalletClient for frontend wallet integration

## Core Projects

<!-- ### Project 0: [Payment Systems](payment-systems/README.md)
**Duration**: 4-6 hours | **Difficulty**: Intermediate

Build a complete payment processing infrastructure supporting various payment scenarios.

**What You'll Build**:
- One-time payment processing
- Invoice generation and management
- Recurring subscription system
- Refund processing
- Payment notifications with webhooks
- Both backend (merchant) and frontend (customer) implementations

**Key Features**:
- Multiple payment types (one-time, subscription, invoice)
- Automated subscription renewals
- Payment history and receipts
- Refund workflow
- Webhook integration for notifications
- Multi-currency display support

**SDK Components Used**:
- Transaction building with fee calculation
- Payment request protocols (BRC-29 style)
- WalletClient for customer payments
- Automated payment processing

**Why Start Here**: Understanding payment flows is fundamental to most blockchain applications. This project teaches core patterns you'll use in all other projects.

--- -->

### Project 1: [Crowdfunding Platform](crowdfunding-platform/README.md)
**Duration**: 4-6 hours | **Difficulty**: Intermediate

Build a complete token-based crowdfunding platform with escrow mechanisms and automated payouts.

**What You'll Build**:
- Token issuance for campaign shares
- Escrow smart contracts for fund security
- Automated payout logic when goals are met
- Campaign dashboard and tracking
- Both backend (custodial) and frontend (WalletClient) implementations

**Key Features**:
- Campaign creation and management
- Token distribution to backers
- Goal tracking and automated payouts
- Refund mechanisms for failed campaigns
- Multi-party escrow patterns

**SDK Components Used**:
- Transaction building with automatic fee handling
- Token template creation
- Escrow script patterns
- BEEF transaction bundles

---

<!-- ### Project 2: [Asset Tokenization System](asset-tokenization/README.md)
**Duration**: 4-6 hours | **Difficulty**: Intermediate

Create a production-ready system for representing real-world assets on the BSV blockchain.

**What You'll Build**:
- Asset registration and tokenization
- Ownership transfer protocols
- Compliance metadata embedding
- Transfer history and provenance tracking
- Both backend and frontend implementations

**Key Features**:
- Real estate, commodities, or certificate tokenization
- Custom business logic with script templates
- Compliance workflow integration
- Ownership verification and transfer
- Metadata standards (BRC integration)

**SDK Components Used**:
- Custom script templates
- Transaction metadata embedding
- Certificate protocols (BRC-56)
- SPV verification for lightweight clients

--- -->

<!-- ### Project 3: [Supply Chain Digital Passports](supply-chain-passports/README.md)
**Duration**: 4-6 hours | **Difficulty**: Intermediate

Implement a complete supply chain tracking system with immutable product passports.

**What You'll Build**:
- Product lifecycle tracking from manufacture to consumer
- Multi-party verification workflows
- Immutable documentation and certification storage
- QR code integration for physical products
- Both enterprise backend and consumer frontend implementations

**Key Features**:
- Product registration and identity creation
- Multi-signature verification from supply chain participants
- Chain of custody tracking
- Quality assurance and compliance documentation
- Overlay network integration for efficient data organization

**SDK Components Used**:
- Multi-signature script templates
- Overlay network patterns
- SPV verification
- BRC standards for interoperability

--- -->

<!-- ## Supporting Modules

These foundational modules provide the technical knowledge needed for the projects above. You can study them in parallel with the projects or reference them as needed.

**Duration**: 1.5 hours

Deep dive into core BSV blockchain building blocks: transaction structure, inputs/outputs, scripts, SIGHASH types, and UTXO management.

### [Transaction Building](transaction-building/README.md)
**Duration**: 2 hours

Master complex transaction construction: multi-input transactions, batch payments, fee optimization, change management, and BEEF format.

### [Script Templates](script-templates/README.md)
**Duration**: 2 hours

Learn Bitcoin Script and create custom locking conditions: P2PKH, multi-sig, hash-locked contracts, time-locks, and custom templates.

### [SPV & Verification](spv-verification/README.md)
**Duration**: 1.5 hours

Implement Simplified Payment Verification for trustless applications: merkle proofs, block header validation, and SPV client development.

### [BRC Standards](brc-standards/README.md)
**Duration**: 2 hours

Implement Bitcoin Request for Comments standards: BRC-42 (key derivation), BRC-43 (security levels), BRC-29 (payments), BRC-56 (certificates). -->

## Development Paradigms

Throughout this path, you'll learn two essential approaches to BSV application development:

### Backend/Service Development
**When to Use**: Internal business systems, automated services, custody solutions

- Application controls wallets and manages keys
- Server-side transaction creation and signing
- SDK handles all transaction mechanics automatically
- Use cases: payment processors, tokenization services, supply chain backends, enterprise wallets

### Frontend Integration Development
**When to Use**: User-facing dApps, consumer applications

- Users control their own wallets via MetaNet Desktop Wallet
- Browser-based transaction signing through WalletClient
- SDK's WalletClient component handles wallet communication
- Use cases: crowdfunding platforms, marketplace frontends, consumer verification apps

**Reference**: [Wallet Toolbox Documentation](https://fast.brc.dev/) and [BRC Standards](https://hub.bsvblockchain.org/brc)

---

## SDK-First Philosophy

All four projects emphasize using the **BSV SDK's pre-built components** rather than manual low-level configuration:

✅ **DO**: Use SDK's built-in transaction methods that handle broadcasting, fees, and UTXO management automatically

✅ **DO**: Use WalletClient for frontend wallet integration with MetaNet Desktop Wallet

✅ **DO**: Let the SDK abstract away infrastructure details

❌ **DON'T**: Manually configure ARC broadcasters, fee calculators, or UTXO managers as primary methods

❌ **DON'T**: Use external APIs (like WhatsOnChain) when SDK provides built-in methods

❌ **DON'T**: Implement low-level protocol details that the SDK handles for you

The SDK provides **ready-to-use, pre-configured components** that let you focus on building applications, not blockchain infrastructure.

---

## Learning Path Completion

After completing this path, you will have:
- ✅ Built a production-ready, deployable BSV application
<!-- - ✅ Implemented complete payment infrastructure (one-time, subscriptions, invoices, refunds) -->
- ✅ Implemented both backend (custodial) and frontend (user-wallet) architectures
- ✅ Mastered the BSV SDK's pre-built components and patterns
- ✅ Created token systems, escrow mechanisms, and multi-party workflows
- ✅ Integrated WalletClient for frontend wallet communication
<!-- - ✅ Implemented SPV verification for lightweight clients -->
<!-- - ✅ Applied BRC standards for interoperability -->
- ✅ Developed a real-world blockchain solution you can adapt for your own projects


## Real-World Applications

The project in this path directly prepares you to build:

<!-- **From Payment Systems:**
- E-commerce payment processing
- Subscription and membership platforms
- Professional invoicing systems
- SaaS billing infrastructure
- Marketplace payment handling
- Digital goods purchasing -->

**From Crowdfunding Platform:**
- Campaign funding systems
- Token-based investment platforms
- Donation and grant management
- Community funding applications

<!-- **From Asset Tokenization:**
- Real estate tokenization
- Commodity trading platforms
- Certificate and credential systems
- Ownership transfer protocols -->

<!-- **From Supply Chain Passports:**
- Product authentication systems
- Supply chain transparency platforms
- Quality assurance tracking
- Multi-party verification workflows -->

## Getting Help

- Check the [SDK Components](../../sdk-components/README.md) for detailed API reference
- Browse [Code Features](../../code-features/README.md) for working examples
- Reference [Wallet Toolbox Documentation](https://fast.brc.dev/) for WalletClient API
- Reference [BRC Standards](https://hub.bsvblockchain.org/brc) for protocol specifications
- Visit [BSV Skills Center](https://hub.bsvblockchain.org/bsv-skills-center) for additional resources
- Download [MetaNet Desktop Wallet](https://desktop.bsvb.tech/) for testing frontend integration
- Join BSV developer communities for peer support

## Estimated Time

**Core Project**: 4-6 hours total
<!-- - Payment Systems: 4-6 hours -->
- Crowdfunding Platform: 4-6 hours
<!-- - Asset Tokenization System: 4-6 hours -->
<!-- - Supply Chain Digital Passports: 4-6 hours -->

<!-- **Supporting Modules**: 13-17 hours total (can be studied in parallel with projects)
- BSV Primitives: 2-3 hours
- Transaction Building: 3-4 hours
- Script Templates: 3-4 hours
- SPV Verification: 3-4 hours
- BRC Standards: 1.5-2 hours -->

**Total Path Duration**: 4-6 hours depending on depth and customization

This is a hands-on learning path - you'll spend most of your time writing and deploying real code, not just reading documentation.

---

## Recommended Learning Path

<!-- 1. **Start with Payment Systems** - Learn fundamental payment flows that apply to all blockchain applications
2. **Choose your next project** based on your interests (Crowdfunding, Tokenization, or Supply Chain)
3. **Reference supporting modules** as needed when you encounter new concepts
4. **Build both implementations** (backend and frontend) for each project
5. **Customize and deploy** your own version with unique features
6. **Move to the next project** and repeat the process -->

1. **Start with Crowdfunding Platform** - Learn token systems, escrow mechanisms, and automated payouts
2. **Build both implementations** (backend and frontend) for the project
3. **Customize and deploy** your own version with unique features

Ready to build production BSV applications? Start with [**Crowdfunding Platform**](crowdfunding-platform/README.md)!
<!-- to learn the fundamentals, then move to [**Crowdfunding Platform**](crowdfunding-platform/README.md), [**Asset Tokenization**](asset-tokenization/README.md), or [**Supply Chain Passports**](supply-chain-passports/README.md)! -->
