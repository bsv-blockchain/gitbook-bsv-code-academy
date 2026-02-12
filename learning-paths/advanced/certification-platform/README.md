# Project: Certification Platform

**Whitelabel BSV Certificate Issuance and Gated Access**

Build a self-hosted certification platform that issues signed, encrypted certificates to users via the BSV SDK and gates protected content behind server-verified certificate authentication. This project demonstrates BSV certificate lifecycle (BRC-52/53), mutual authentication (BRC-31), field-level encryption with ECDH key agreement, and the direct acquisition protocol.

> **Complete Code Repository**: [https://github.com/bsv-blockchain-demos/certification-platform](https://github.com/bsv-blockchain-demos/certification-platform)
>
> This guide references the full production implementation. Clone the repository to follow along with working code examples.

---

## What You'll Build

A production-ready certification platform featuring:

- Self-hosted certificate issuance using `MasterCertificate`
- Encrypted certificate fields with ECDH key agreement (BRC-42)
- Direct acquisition protocol for wallet storage
- Mutual authentication via `AuthFetch` and Express auth middleware (BRC-31)
- Certificate-gated dashboard with server-side verification
- Certificate revocation and session management
- Frontend and backend implementation patterns

---

## Learning Objectives

By completing this project, you will learn:

- **Certificate Model (BRC-52)** - Signed, encrypted certificate structure with field-level encryption
- **Certificate Verification (BRC-53)** - Decrypting and verifying certificate fields server-side
- **Mutual Authentication (BRC-31)** - `AuthFetch` + Express auth middleware session protocol
- **Key Derivation (BRC-42)** - ECDH-based field encryption keyrings
- **Wallet Types** - ProtoWallet, Wallet, and WalletClient for different roles
- **Direct Acquisition** - Storing pre-signed certificates in browser wallets
- **Session Management** - `SessionManager` for authenticated connections

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│    BSV Wallet Extension (User)          │
│  - Identity key management              │
│  - Certificate storage                  │
│  - Auth protocol negotiation            │
└─────────────────┬───────────────────────┘
                  │
                  │ WalletClient
                  │
┌─────────────────▼───────────────────────┐
│       Frontend (Next.js)                │
│  - WalletClient for identity            │
│  - Certificate acquisition UI           │
│  - AuthFetch for gated routes           │
│  - Dashboard with protected content     │
└─────────────────┬───────────────────────┘
                  │
                  │ HTTP / AuthFetch
                  │
┌─────────────────▼───────────────────────┐
│       Backend (Express.js)              │
│  - /api/info — certifier discovery      │
│  - /api/certify — certificate issuance  │
│  - Auth middleware (BRC-31)             │
│  - /protected/dashboard — gated route   │
│  - /relinquish — revocation             │
└─────────────────┬───────────────────────┘
                  │
                  │ SDK Components
                  │
┌─────────────────▼───────────────────────┐
│          BSV SDK Stack                  │
│  - MasterCertificate (issuance)         │
│  - VerifiableCertificate (verification) │
│  - ProtoWallet (signing)                │
│  - Wallet (auth middleware)             │
│  - SessionManager (sessions)            │
│  - KeyDeriver (encryption)              │
└─────────────────────────────────────────┘
```

---

## Implementation Guides

This project is split into two implementation guides:

### [Frontend Application](./frontend.md)

Learn how to build the browser-side certification interface:

- Connecting to user wallets with WalletClient
- Discovering certifier info from the backend
- Checking for existing certificates
- Acquiring certificates via direct protocol
- Revoking certificates from the wallet
- Accessing gated content with AuthFetch

### [Backend Server](./backend.md)

Learn how to implement the server-side certification engine:

- Server identity and key derivation
- ProtoWallet for certificate signing
- Full Wallet setup for auth middleware
- MasterCertificate issuance with field encryption
- Auth middleware configuration and session management
- Certificate verification and field decryption
- Protected route gating

---

## Key Concepts

### Self-Hosted Certifier

Unlike systems that depend on external certificate authorities, this platform IS the certifier. Your server holds the private key, signs certificates, and controls the trust model:

```
Traditional PKI:
User → External CA → Trust Chain → Verification

BSV Certification:
User → Your Server (certifier) → Signed Certificate → Wallet Storage
```

**Benefits:**
- No third-party dependency
- Full control over issuance rules
- Custom certificate fields
- Whitelabel deployment ready

### Certificate Lifecycle

```
1. DISCOVERY    GET /api/info → { certifierPublicKey, certificateType }
2. ISSUANCE     POST /api/certify → MasterCertificate (encrypted + signed)
3. STORAGE      wallet.acquireCertificate({ protocol: 'direct' })
4. GATED ACCESS AuthFetch → auth middleware → certificate verification
5. REVOCATION   wallet.relinquishCertificate() + POST /relinquish
```

### Field-Level Encryption

Certificate fields are never stored in plaintext. Each field is encrypted with a unique key derived via ECDH between the certifier and subject:

```
Certifier Private Key + Subject Public Key
              │
              ▼
        ECDH Shared Secret (per field)
              │
     ┌────────┴────────┐
     ▼                 ▼
Field Encryption    Keyring Entry
(AES encrypt        (encrypt shared key
 field value)        TO subject's pubkey)
```

Only the certificate subject (user) or the certifier (server) can decrypt the fields.

### Wallet Architecture

Three wallet tiers serve different purposes:

| Wallet | Location | Purpose |
|--------|----------|---------|
| `ProtoWallet` | Backend | Lightweight — signs certificates (no storage needed) |
| `Wallet` | Backend | Full wallet — auth middleware needs ECDH, sessions, decryption |
| `WalletClient` | Frontend | Browser proxy — connects to wallet extension via message passing |

### Direct Acquisition Protocol

This system uses `'direct'` acquisition because the backend IS the certifier. No external handshake is needed — the certificate is already signed and complete when it reaches the frontend:

```typescript
await wallet.acquireCertificate({
  acquisitionProtocol: 'direct',   // No external certifier URL
  keyringRevealer: 'certifier',    // Keyring encrypted by certifier
  ...certData                       // Pre-signed certificate data
})
```

---

## SDK Stack

Three npm packages form the BSV layer:

| Package | Version | Location | Purpose |
|---------|---------|----------|---------|
| `@bsv/sdk` | 1.10.1+ | Frontend + Backend | Core cryptography, certificates, AuthFetch, WalletClient |
| `@bsv/wallet-toolbox-client` | 1.7.18+ | Backend only | Full server wallet with storage and chain services |
| `@bsv/auth-express-middleware` | 1.2.3+ | Backend only | Express middleware for mutual authentication |

---

## Getting Started

### Quick Start

```bash
# Clone the complete working implementation
git clone https://github.com/bsv-blockchain-demos/certification-platform.git
cd certification-platform

# Start backend
cd backend
npm install
npm run dev
# Running on http://localhost:3002

# In another terminal, start frontend
cd frontend
npm install
npm run dev
# Running on http://localhost:3000
```

### Prerequisites

- Node.js 18+
- BSV Wallet Extension (e.g., Yours Wallet)
- TypeScript 4.9+
- A wallet storage service URL (for backend Wallet initialization)

### Choose Your Learning Path

1. **[Backend Server](./backend.md)** — Start here to understand the certification engine
2. **[Frontend Application](./frontend.md)** — Build the user-facing interface

Both guides use the BSV SDK and reference the complete working code.

---

## Project Structure

```
certification-platform/
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx              # Root layout with WalletProvider
│   │   │   ├── page.tsx                # Certificate issuance page
│   │   │   └── dashboard/
│   │   │       └── page.tsx            # Protected dashboard (AuthFetch)
│   │   ├── context/
│   │   │   └── WalletContext.tsx        # WalletClient provider
│   │   ├── components/
│   │   │   └── Nav.tsx                 # Navigation bar
│   │   └── constants.ts                # API URL configuration
│   ├── package.json
│   └── next.config.mjs
│
├── backend/
│   ├── src/
│   │   ├── server.ts                   # Express server + routes
│   │   └── wallet.ts                   # Wallet setup + key management
│   ├── package.json
│   └── tsconfig.json
│
└── README.md
```

---

## Use Cases

### Access Control
- Gated content areas (dashboards, premium features)
- Role-based access with custom certificate fields
- Membership verification systems

### Identity Verification
- KYC/AML compliance certificates
- Professional credentials (certifications, licenses)
- Organization membership proofs

### Whitelabel Deployment
- Custom branding and certificate fields
- Self-hosted infrastructure
- Full control over issuance policy
- No dependency on external certifiers

### API Access Management
- Certificate-gated API endpoints
- Machine-to-machine authentication
- Revocable access tokens

---

## Troubleshooting

**"Wallet Not Found"**
- Install a BSV wallet extension (e.g., Yours Wallet)
- Unlock the wallet before using the frontend

**"Failed to get certificate from server"**
- Ensure the backend is running on the correct port
- Check `NEXT_PUBLIC_API_URL` environment variable

**"Certificate Required" on dashboard**
- Acquire a certificate first on the main certify page
- Ensure the certificate is stored in the wallet (check `listCertificates`)

**"Access denied" after acquiring certificate**
- The auth middleware may not have processed the certificate yet
- Check backend logs for `onCertificatesReceived` output
- Verify the certificate type and certifier match the middleware config

---

## Project Completion Checklist

- [ ] Backend serves certifier info via `/api/info`
- [ ] Backend issues encrypted, signed certificates via `/api/certify`
- [ ] Frontend discovers certifier and checks existing certificates
- [ ] Frontend acquires certificates via direct protocol
- [ ] Certificates stored in browser wallet with encrypted fields
- [ ] AuthFetch authenticates against auth middleware
- [ ] Server decrypts and verifies certificate fields
- [ ] Protected dashboard accessible only to certificate holders
- [ ] Certificate revocation removes wallet cert and server session
- [ ] Full 4-phase flow works end-to-end

---

## Summary

This project demonstrates a complete BSV certification system combining:

- **Self-hosted certificate issuance** (MasterCertificate + ProtoWallet)
- **Field-level encryption** (ECDH key agreement + keyrings)
- **Mutual authentication** (AuthFetch + Express auth middleware)
- **Server-side verification** (VerifiableCertificate field decryption)
- **Direct acquisition** (pre-signed certificates stored in browser wallet)
- **Session management** (SessionManager for authenticated connections)
- **Certificate revocation** (wallet removal + server session cleanup)

These patterns form the foundation for building access-controlled applications, identity verification systems, and whitelabel certification platforms on BSV.

---

## Related Resources

### BRC Standards
- [BRC-31: Auth Protocol](https://brc.dev/31)
- [BRC-42: Key Derivation Protocol](https://brc.dev/42)
- [BRC-43: Security Levels](https://brc.dev/43)
- [BRC-52: Certificate Creation](https://brc.dev/52)
- [BRC-53: Certificate Verification](https://brc.dev/53)
- [BRC-56: Wallet Wire Protocol](https://brc.dev/56)

### Documentation
- [BSV SDK Documentation](https://docs.bsvblockchain.org/)
- [BSV Desktop Wallet](https://desktop.bsvb.tech/)

### Prerequisites
- [WalletClient Integration](../../beginner/wallet-client-integration/README.md)
- [Intermediate Path](../../intermediate/README.md)
- [Beginner Path](../../beginner/README.md)
