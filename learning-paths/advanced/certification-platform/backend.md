# Certification Platform - Backend Guide

**Building the Server-Side Certification Engine with BSV SDK**

This guide covers backend patterns for implementing certificate issuance, field encryption, mutual authentication middleware, and certificate-gated routes using the BSV SDK, wallet-toolbox-client, and auth-express-middleware.

> **Complete Code Repository**: [https://github.com/bsv-blockchain-demos/certification-platform](https://github.com/bsv-blockchain-demos/certification-platform)
>
> All code examples in this guide are taken from the working implementation in the `backend/` directory.

---

## Table of Contents

1. [Setup](#setup)
2. [Server Identity & Key Architecture](#server-identity--key-architecture)
3. [Wallet Configuration](#wallet-configuration)
4. [Certificate Issuance](#certificate-issuance)
5. [Auth Middleware & Sessions](#auth-middleware--sessions)
6. [Certificate Verification & Field Decryption](#certificate-verification--field-decryption)
7. [Protected Routes](#protected-routes)
8. [Certificate Revocation](#certificate-revocation)
9. [Complete Server](#complete-server)

---

## Setup

### Installation

```bash
npm install @bsv/sdk @bsv/wallet-toolbox-client @bsv/auth-express-middleware express cors dotenv
npm install -D @types/node @types/express typescript
```

### Project Structure

```
backend/
├── src/
│   ├── server.ts                   # Express server + all routes
│   └── wallet.ts                   # Key management + wallet setup
├── .env
├── package.json
└── tsconfig.json
```

### Environment Configuration

Create `.env`:

```env
SERVER_PRIVATE_KEY=bcc56b658e5b8660ceba47f323e8a77c4794ab9c76f2bd0082a056c723980049
BSV_NETWORK=test
WALLET_STORAGE_URL=https://storage.babbage.systems
PORT=3002
```

| Variable | Purpose |
|----------|---------|
| `SERVER_PRIVATE_KEY` | 32-byte hex private key — this IS the server's identity |
| `BSV_NETWORK` | `'test'` or `'main'` — determines chain for wallet services |
| `WALLET_STORAGE_URL` | Remote storage endpoint for the full Wallet |
| `PORT` | Express server port |

---

## Server Identity & Key Architecture

### Pattern 1: Deriving the Certifier Identity

Every participant in the BSV certificate system has an identity derived from an elliptic curve key pair (secp256k1). The server's private key generates its public identity:

> **See full implementation**: `backend/src/wallet.ts`

```typescript
// src/wallet.ts

import { PrivateKey, KeyDeriver, ProtoWallet } from '@bsv/sdk'

const SERVER_PRIVATE_KEY = process.env.SERVER_PRIVATE_KEY
  ?? 'bcc56b658e5b8660ceba47f323e8a77c4794ab9c76f2bd0082a056c723980049'

/**
 * Derive the certifier's public key from the server's private key
 * This IS the server's identity — it appears in every certificate
 */
export function getCertifierPublicKey(): string {
  return new PrivateKey(SERVER_PRIVATE_KEY, 'hex')
    .toPublicKey()
    .toString()
}
```

**Where the Certifier Public Key Appears:**

```
Certificate.certifier         → identifies who signed the cert
Auth middleware config         → which certifiers to accept
Frontend listCertificates()   → filter certs from this server
/api/info response            → client discovery
```

**Key Derivation Chain:**

```
SERVER_PRIVATE_KEY (32 bytes hex)
         │
         ▼
    PrivateKey
         │
    ┌────┴─────────────────────────────────────────┐
    │                                              │
    ▼                                              ▼
 .toPublicKey().toString()              new KeyDeriver(privateKey)
    │                                              │
    ▼                                              ▼
 Certifier Public Key                    keyDeriver.identityKey
 (33 bytes compressed hex)              (same public key, used by
  "02a1b2c3..."                          wallet toolbox internally)
```

### Certificate Type

The certificate type is a base64-encoded string that categorizes the certificate. The SDK's `Utils` class handles encoding:

```typescript
import { Utils } from '@bsv/sdk'

const CERTIFICATE_TYPE = Utils.toBase64(
  Utils.toArray('certification', 'utf8')
)
// = "Y2VydGlmaWNhdGlvbg=="
```

`Utils.toArray(str, 'utf8')` converts the string to a byte array. `Utils.toBase64(bytes)` encodes it. This is the standard way to create certificate type identifiers in the BSV SDK.

---

## Wallet Configuration

### Pattern 2: Two Wallet Tiers

The backend uses two different wallet types for different purposes:

**ProtoWallet — Certificate Signing**

Lightweight wallet created from just a `PrivateKey`. No storage, no network. Used exclusively for `MasterCertificate.issueCertificateForSubject()`:

```typescript
export function getCertifierWallet(): ProtoWallet {
  return new ProtoWallet(new PrivateKey(SERVER_PRIVATE_KEY, 'hex'))
}
```

**Why ProtoWallet?** `MasterCertificate.issueCertificateForSubject()` only needs cryptographic operations (key derivation, encryption, signing). A `ProtoWallet` is sufficient and avoids the overhead of storage initialization.

**Full Wallet — Auth Middleware**

The auth middleware requires a full `WalletInterface` because it needs to decrypt certificate fields, manage session nonces, and respond to the client's auth protocol:

```typescript
import { Wallet, WalletSigner, WalletStorageManager, StorageClient, Services }
  from '@bsv/wallet-toolbox-client'
import { PrivateKey, KeyDeriver, WalletInterface } from '@bsv/sdk'

const BSV_NETWORK = process.env.BSV_NETWORK || 'test'
const WALLET_STORAGE_URL = process.env.WALLET_STORAGE_URL
  || 'https://storage.babbage.systems'

export async function getWallet(): Promise<WalletInterface> {
  const chain = BSV_NETWORK !== 'test' ? 'main' : 'test'
  const keyDeriver = new KeyDeriver(new PrivateKey(SERVER_PRIVATE_KEY, 'hex'))
  const storageManager = new WalletStorageManager(keyDeriver.identityKey)
  const signer = new WalletSigner(chain, keyDeriver, storageManager)
  const services = new Services(chain)
  const wallet = new Wallet(signer, services)
  const client = new StorageClient(wallet, WALLET_STORAGE_URL)
  await client.makeAvailable()
  await storageManager.addWalletStorageProvider(client)
  return wallet
}
```

**Construction Chain:**

```
PrivateKey
    │
    ▼
KeyDeriver ──────────────────────────┐
    │                                │
    ▼                                ▼
WalletStorageManager          WalletSigner
    │                           │    │
    │                           │    ▼
    │                           │  Services (chain='test'|'main')
    │                           │    │
    │                           ▼    ▼
    │                        Wallet
    │                           │
    ▼                           ▼
StorageClient ◀──── wallet reference
    │
    ▼
storageManager.addWalletStorageProvider(client)
    │
    ▼
  Ready — wallet can now read/write to remote storage
```

### Why Two Wallets?

| Operation | Needs Storage? | Needs Network? | Wallet |
|-----------|---------------|----------------|--------|
| Issue certificates | No | No | ProtoWallet |
| Auth middleware (ECDH, sessions) | Yes | Yes | Full Wallet |
| Decrypt certificate fields | Yes | No | Full Wallet |

---

## Certificate Issuance

### Pattern 3: MasterCertificate.issueCertificateForSubject()

The core issuance method creates a complete, encrypted, signed certificate for a given subject:

```typescript
import { MasterCertificate, Utils } from '@bsv/sdk'

const CERTIFICATE_TYPE = Utils.toBase64(
  Utils.toArray('certification', 'utf8')
)

// Inside the /api/certify route handler:
const certifierWallet = getCertifierWallet()  // ProtoWallet

const masterCert = await MasterCertificate.issueCertificateForSubject(
  certifierWallet,
  identityKey,                                // Subject's public key (from request)
  {
    certified: 'true',
    timestamp: Math.floor(Date.now() / 1000).toString()
  },
  CERTIFICATE_TYPE,
  async () => '00'.repeat(32) + '.0'          // Dummy revocation outpoint
)
```

**Parameters:**

| Parameter | Type | Value |
|-----------|------|-------|
| `certifierWallet` | `ProtoWallet` | Server's lightweight wallet |
| `subjectPublicKey` | `string` | User's identity key from request body |
| `fields` | `Record<string, string>` | Plaintext fields to encrypt |
| `certificateType` | `string` | `"Y2VydGlmaWNhdGlvbg=="` |
| `getRevocationOutpoint` | `() => Promise<string>` | Returns `"000...0.0"` |

**What Happens Internally:**

```
issueCertificateForSubject()
│
├── 1. Generate a unique serialNumber
│
├── 2. For each field { certified, timestamp }:
│   │
│   ├── a. Derive field-specific symmetric key using:
│   │      ECDH(certifierPrivateKey, subjectPublicKey)
│   │      + field name as key derivation context
│   │
│   ├── b. Encrypt field value with symmetric key
│   │      → fields.certified = "<encrypted>"
│   │      → fields.timestamp = "<encrypted>"
│   │
│   └── c. Create keyring entry for this field:
│          Encrypt the symmetric key TO the subject's public key
│          → keyringForSubject.certified = "<encrypted-key>"
│          → keyringForSubject.timestamp = "<encrypted-key>"
│
├── 3. Call getRevocationOutpoint() → "000...000.0"
│
├── 4. Construct the certificate object
│
├── 5. Sign with certifier's private key
│      → signature = ECDSA(certifierPrivKey, certHash)
│
└── 6. Return MasterCertificate instance
```

**Returning the Certificate:**

```typescript
// Return certificate data to frontend
res.json({
  type:                masterCert.type,
  serialNumber:        masterCert.serialNumber,
  subject:             masterCert.subject,
  certifier:           masterCert.certifier,
  revocationOutpoint:  masterCert.revocationOutpoint,
  fields:              masterCert.fields,              // encrypted
  signature:           masterCert.signature,
  keyringForSubject:   masterCert.masterKeyring         // encrypted keys
})
```

### Field Encryption Model

Certificate fields use a two-layer encryption scheme based on ECDH key agreement:

```
               CERTIFIER                          SUBJECT (user)
             (server key)                        (wallet key)
                  │                                   │
                  │          ECDH Key Agreement       │
                  │◀─────────────────────────────────▶│
                  │                                   │
                  ▼                                   │
         Shared Secret (per field)                    │
                  │                                   │
         ┌────────┴────────┐                          │
         │                 │                          │
         ▼                 ▼                          │
  Field Encryption   Keyring Entry                   │
         │                 │                          │
         │   AES(sharedKey, fieldValue)              │
         │   → encrypted field                       │
         │                                            │
         │   AES(subjectKey, sharedKey)              │
         │   → keyring entry                         │
         │   (subject can recover sharedKey)         │
         │                                            │
         ▼                                            ▼
  Certificate.fields              Certificate.keyringForSubject
```

**Why Two Layers?**
1. **Field values** are encrypted with a shared key from ECDH
2. **The shared key** is encrypted in the keyring, targeted at the subject's public key

This means only the subject (user) or the certifier (server) can decrypt the fields.

### Revocation Outpoint Format

The `revocationOutpoint` must follow the format `<txid>.<outputIndex>`:

```typescript
// VALID — 64-char hex txid + "." + output index
async () => '00'.repeat(32) + '.0'     // "000...000.0"

// INVALID — the SDK will throw
async () => '00'.repeat(32)            // Missing ".outputIndex"
```

In production, this would point to a real UTXO. Spending that UTXO revokes the certificate.

---

## Auth Middleware & Sessions

### Pattern 4: createAuthMiddleware Configuration

`@bsv/auth-express-middleware` implements mutual authentication between the client (browser wallet) and server. It intercepts requests on protected routes:

```typescript
import { createAuthMiddleware, AuthRequest } from '@bsv/auth-express-middleware'
import { SessionManager } from '@bsv/sdk'

const sessionManager = new SessionManager()

const authMiddleware = createAuthMiddleware({
  wallet,                            // Full WalletInterface (not ProtoWallet)
  certificatesToRequest: {
    certifiers: [CERTIFIER_PUBLIC_KEY],
    types: {
      [CERTIFICATE_TYPE]: ['certified', 'timestamp']
    }
  },
  sessionManager,
  onCertificatesReceived,            // Callback when client presents certs
  logger: console,
  logLevel: 'debug'
})

// Apply to all routes below this line
app.use(authMiddleware)
```

**Configuration Parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `wallet` | `WalletInterface` | Server's wallet — ECDH, decryption, session signing |
| `certificatesToRequest.certifiers` | `string[]` | Only accept certs from these public keys |
| `certificatesToRequest.types` | `Record<string, string[]>` | Certificate types and which fields to request |
| `sessionManager` | `SessionManager` | Manages auth session state |
| `onCertificatesReceived` | Callback | Called when client presents certificates |
| `logger` | `Console` | Logging interface |
| `logLevel` | `string` | `'debug'`, `'info'`, `'warn'`, `'error'` |

### Session Negotiation Protocol

```
AuthFetch (client)                    Auth Middleware (server)
       │                                      │
  1.   │  Initial request                     │
       │─────────────────────────────────────▶│
       │                                      │  No session found
       │  401 + session challenge             │
       │  (nonce, server identity)            │
       │◀─────────────────────────────────────│
       │                                      │
  2.   │  Auth response                       │
       │  (signed nonce, identity,            │
       │   certificates + keyrings)           │
       │─────────────────────────────────────▶│
       │                                      │  Verify signature
       │                                      │  Extract certificates
       │                                      │  Call onCertificatesReceived()
       │                                      │  Create session
       │                                      │
       │  200 + session token                 │
       │◀─────────────────────────────────────│
       │                                      │
  3.   │  Subsequent requests                 │
       │  (session token in header)           │
       │─────────────────────────────────────▶│
       │                                      │  Validate session
       │                                      │  req.auth.identityKey = "03..."
       │                                      │  next()
```

### SessionManager

`SessionManager` from `@bsv/sdk` manages the mapping between client identity keys and session tokens:

```typescript
const sessionManager = new SessionManager()

// Used internally by auth middleware:
// sessionManager.createSession(identityKey) → session token
// sessionManager.getSession(identityKey)    → session object
// sessionManager.removeSession(session)     → void

// Manual removal on revocation:
const session = sessionManager.getSession(identityKey)
if (session != null) {
  sessionManager.removeSession(session)
}
```

---

## Certificate Verification & Field Decryption

### Pattern 5: VerifiableCertificate

When the auth middleware receives certificates from a client, the `onCertificatesReceived` callback decrypts and verifies the certificate fields:

```typescript
import { VerifiableCertificate } from '@bsv/sdk'

async function decryptFields(cert: any): Promise<void> {
  if (cert.keyring != null && cert.fields != null && cert.certifier != null) {
    const verifiableCert = VerifiableCertificate.fromCertificate(cert, cert.keyring)
    const decryptedFields = await verifiableCert.decryptFields(wallet)
    cert.fields = decryptedFields
  }
}
```

**Decryption Flow:**

```
cert.keyring                cert.fields
(encrypted keys)            (encrypted values)
       │                          │
       ▼                          │
VerifiableCertificate             │
  .fromCertificate(cert, keyring) │
       │                          │
       ▼                          │
  .decryptFields(wallet)          │
       │                          │
       ├── For each field:        │
       │   │                      │
       │   ├── 1. Decrypt keyring entry using wallet's         │
       │   │      private key (ECDH with subject's pubkey)     │
       │   │      → recovers the field's symmetric key         │
       │   │                                                   │
       │   ├── 2. Use symmetric key to decrypt field value ◀───┘
       │   │      → plaintext value
       │   │
       │   └── 3. Store in decryptedFields map
       │
       └── Return { certified: 'true', timestamp: '1705312200' }
```

### onCertificatesReceived Callback

This callback fires when a client presents certificates during authentication:

```typescript
function onCertificatesReceived(
  senderPublicKey: string,
  certs: any[],
  req: AuthRequest,
  res: Response,
  next: NextFunction
): void {
  for (const cert of certs) {
    decryptFields(cert)
      .then(() => {
        // Fields are now decrypted — check values
        if (cert.fields?.certified === 'true') {
          verifier.setVerified(senderPublicKey)
        }
      })
      .catch((error) => {
        console.error('Error decrypting fields:', error)
      })
  }
  next()  // Continue request processing
}
```

### Why the Server Can Decrypt

The server's wallet holds the same private key used as the certifier when issuing the certificate. Since field encryption keys are derived via ECDH between certifier and subject, the server has one side of the ECDH:

```
Issuance:   ECDH(certifierPrivKey, subjectPubKey) → sharedKey → encrypt fields
Decryption: ECDH(certifierPrivKey, subjectPubKey) → sharedKey → decrypt fields
                    ▲
                    │
              Same private key on both sides
              (server issued it, server verifies it)
```

### Verification Tracking

Use a simple in-memory map to track which users have verified certificates:

```typescript
class CertificateVerifier {
  private verified = new Map<string, boolean>()

  setVerified(identityKey: string): void {
    this.verified.set(identityKey, true)
  }

  isVerified(identityKey: string): boolean {
    return this.verified.get(identityKey) === true
  }

  clear(identityKey: string): void {
    this.verified.delete(identityKey)
  }
}

const verifier = new CertificateVerifier()
```

---

## Protected Routes

### Pattern 6: Certificate-Gated Endpoints

Routes placed after the auth middleware are automatically protected. The middleware adds `req.auth.identityKey` to authenticated requests:

```typescript
// Protected route — requires valid certificate
app.get('/protected/dashboard', (req: AuthRequest, res) => {
  const identityKey = req.auth?.identityKey

  if (!identityKey || !verifier.isVerified(identityKey)) {
    return res.status(401).json({
      success: false,
      error: 'Certificate not verified'
    })
  }

  res.json({
    success: true,
    data: {
      status: 'verified',
      accessLevel: 'full',
      certifiedAt: new Date().toISOString()
    }
  })
})
```

**Route Organization:**

```
// Public routes — no auth needed
app.get('/api/info', ...)          // Certifier discovery
app.post('/api/certify', ...)      // Certificate issuance

// Auth middleware — applied to all routes below
app.use(authMiddleware)

// Protected routes — require valid certificate
app.get('/protected/dashboard', ...)  // Gated content
app.post('/relinquish', ...)          // Certificate revocation
```

---

## Certificate Revocation

### Pattern 7: Session and Verification Cleanup

When a user revokes their certificate, clean up server-side state:

```typescript
app.post('/relinquish', (req: AuthRequest, res) => {
  const identityKey = req.auth?.identityKey

  if (identityKey) {
    // Remove verification status
    verifier.clear(identityKey)

    // Remove session
    const session = sessionManager.getSession(identityKey)
    if (session != null) {
      sessionManager.removeSession(session)
    }
  }

  res.json({ success: true })
})
```

**Revocation Flow:**

```
Frontend                   Auth Middleware          Backend
   │                            │                     │
   │  wallet.relinquishCert()   │                     │
   │  → removed from wallet     │                     │
   │                            │                     │
   │  AuthFetch                 │                     │
   │  POST /relinquish          │                     │
   │───────────────────────────▶│                     │
   │                            │────────────────────▶│
   │                            │     verifier.clear()│
   │                            │     session.remove()│
   │  200 { success }           │                     │
   │◀─────────────────────────────────────────────────│
```

---

## Complete Server

### Main Server Setup

> **See full implementation**: `backend/src/server.ts`

```typescript
// src/server.ts

import express from 'express'
import cors from 'cors'
import { config } from 'dotenv'
import {
  MasterCertificate,
  VerifiableCertificate,
  SessionManager,
  Utils
} from '@bsv/sdk'
import { createAuthMiddleware, AuthRequest } from '@bsv/auth-express-middleware'
import {
  getCertifierPublicKey,
  getCertifierWallet,
  getWallet
} from './wallet'

config()

const app = express()
const PORT = parseInt(process.env.PORT || '3002')

// Middleware
app.use(cors({ origin: true, credentials: true }))
app.use(express.json())

// Constants
const CERTIFIER_PUBLIC_KEY = getCertifierPublicKey()
const CERTIFICATE_TYPE = Utils.toBase64(
  Utils.toArray('certification', 'utf8')
)

// Verification tracker
class CertificateVerifier {
  private verified = new Map<string, boolean>()
  setVerified(key: string) { this.verified.set(key, true) }
  isVerified(key: string) { return this.verified.get(key) === true }
  clear(key: string) { this.verified.delete(key) }
}
const verifier = new CertificateVerifier()

// ───────────────────────────────────────────
// PUBLIC ROUTES
// ───────────────────────────────────────────

// Certifier discovery
app.get('/api/info', (req, res) => {
  res.json({
    certifierPublicKey: CERTIFIER_PUBLIC_KEY,
    certificateType: CERTIFICATE_TYPE
  })
})

// Certificate issuance
app.post('/api/certify', async (req, res) => {
  try {
    const { identityKey } = req.body

    if (!identityKey) {
      return res.status(400).json({ error: 'identityKey is required' })
    }

    const certifierWallet = getCertifierWallet()

    const masterCert = await MasterCertificate.issueCertificateForSubject(
      certifierWallet,
      identityKey,
      {
        certified: 'true',
        timestamp: Math.floor(Date.now() / 1000).toString()
      },
      CERTIFICATE_TYPE,
      async () => '00'.repeat(32) + '.0'
    )

    res.json({
      type:               masterCert.type,
      serialNumber:       masterCert.serialNumber,
      subject:            masterCert.subject,
      certifier:          masterCert.certifier,
      revocationOutpoint: masterCert.revocationOutpoint,
      fields:             masterCert.fields,
      signature:          masterCert.signature,
      keyringForSubject:  masterCert.masterKeyring
    })
  } catch (error: any) {
    console.error('Certification error:', error)
    res.status(500).json({ error: error.message })
  }
})

// ───────────────────────────────────────────
// AUTH MIDDLEWARE SETUP
// ───────────────────────────────────────────

async function startServer() {
  const wallet = await getWallet()
  const sessionManager = new SessionManager()

  // Field decryption helper
  async function decryptFields(cert: any): Promise<void> {
    if (cert.keyring != null && cert.fields != null && cert.certifier != null) {
      const verifiableCert = VerifiableCertificate.fromCertificate(
        cert,
        cert.keyring
      )
      const decryptedFields = await verifiableCert.decryptFields(wallet)
      cert.fields = decryptedFields
    }
  }

  // Certificate verification callback
  function onCertificatesReceived(
    senderPublicKey: string,
    certs: any[],
    req: AuthRequest,
    res: express.Response,
    next: express.NextFunction
  ): void {
    for (const cert of certs) {
      decryptFields(cert)
        .then(() => {
          if (cert.fields?.certified === 'true') {
            verifier.setVerified(senderPublicKey)
          }
        })
        .catch((error) => {
          console.error('Error decrypting fields:', error)
        })
    }
    next()
  }

  // Create and apply auth middleware
  const authMiddleware = createAuthMiddleware({
    wallet,
    certificatesToRequest: {
      certifiers: [CERTIFIER_PUBLIC_KEY],
      types: {
        [CERTIFICATE_TYPE]: ['certified', 'timestamp']
      }
    },
    sessionManager,
    onCertificatesReceived,
    logger: console,
    logLevel: 'debug'
  })

  app.use(authMiddleware)

  // ─────────────────────────────────────────
  // PROTECTED ROUTES (require valid cert)
  // ─────────────────────────────────────────

  app.get('/protected/dashboard', (req: AuthRequest, res) => {
    const identityKey = req.auth?.identityKey

    if (!identityKey || !verifier.isVerified(identityKey)) {
      return res.status(401).json({
        success: false,
        error: 'Certificate not verified'
      })
    }

    res.json({
      success: true,
      data: {
        status: 'verified',
        accessLevel: 'full',
        certifiedAt: new Date().toISOString()
      }
    })
  })

  app.post('/relinquish', (req: AuthRequest, res) => {
    const identityKey = req.auth?.identityKey

    if (identityKey) {
      verifier.clear(identityKey)
      const session = sessionManager.getSession(identityKey)
      if (session != null) {
        sessionManager.removeSession(session)
      }
    }

    res.json({ success: true })
  })

  // Start listening
  app.listen(PORT, () => {
    console.log(`Certification server running on http://localhost:${PORT}`)
    console.log(`Certifier: ${CERTIFIER_PUBLIC_KEY}`)
    console.log(`Certificate type: ${CERTIFICATE_TYPE}`)
    console.log('\nEndpoints:')
    console.log('  GET  /api/info              — Certifier discovery')
    console.log('  POST /api/certify           — Issue certificate')
    console.log('  GET  /protected/dashboard   — Gated content')
    console.log('  POST /relinquish            — Revoke certificate')
  })
}

startServer().catch(console.error)
```

---

## Important Concepts

### SDK Class Reference

| Class | Import | Purpose |
|-------|--------|---------|
| `PrivateKey` | `@bsv/sdk` | Server identity key |
| `KeyDeriver` | `@bsv/sdk` | Derives child keys from root private key |
| `ProtoWallet` | `@bsv/sdk` | Lightweight wallet for certificate signing |
| `MasterCertificate` | `@bsv/sdk` | Issues encrypted, signed certificates |
| `VerifiableCertificate` | `@bsv/sdk` | Wraps cert + keyring for field decryption |
| `SessionManager` | `@bsv/sdk` | Manages auth session state |
| `Utils` | `@bsv/sdk` | Base64 encoding, byte array conversions |
| `Wallet` | `@bsv/wallet-toolbox-client` | Full wallet implementation |
| `WalletSigner` | `@bsv/wallet-toolbox-client` | Transaction and message signing |
| `WalletStorageManager` | `@bsv/wallet-toolbox-client` | Manages storage providers |
| `StorageClient` | `@bsv/wallet-toolbox-client` | Connects to remote storage |
| `Services` | `@bsv/wallet-toolbox-client` | Network services (chain queries) |
| `createAuthMiddleware` | `@bsv/auth-express-middleware` | Factory for Express auth middleware |
| `AuthRequest` | `@bsv/auth-express-middleware` | Extended Request type with `auth` property |

### Route Architecture

```
Request Flow:

  /api/info ─────────────────────────▶ Public handler
  /api/certify ──────────────────────▶ Public handler

  /protected/* ──▶ Auth Middleware ──▶ Protected handler
  /relinquish ───▶ Auth Middleware ──▶ Protected handler
                        │
                        ├── No session? → 401 challenge
                        ├── Has session? → req.auth.identityKey
                        └── Has certs? → onCertificatesReceived()
```

### Security Considerations

1. **Private Key Storage**: The `SERVER_PRIVATE_KEY` should be stored securely (environment variable, secrets manager) — never committed to source control
2. **CORS**: Configure `origin` to your frontend's domain in production
3. **HTTPS**: Use TLS in production for all endpoints
4. **Revocation Outpoints**: In production, use real UTXOs for certificate revocation tracking
5. **Rate Limiting**: Add rate limits to `/api/certify` to prevent abuse

### BRC Standards Implemented

| BRC | Standard | Implementation |
|-----|----------|---------------|
| BRC-31 | Auth Protocol | `createAuthMiddleware` + `AuthFetch` mutual authentication |
| BRC-42 | Key Derivation | `KeyDeriver` derives child keys for field encryption |
| BRC-43 | Security Levels | Certificate field keys use specific security levels in ECDH |
| BRC-52 | Certificate Creation | `MasterCertificate.issueCertificateForSubject()` |
| BRC-53 | Certificate Verification | `VerifiableCertificate.decryptFields()` |
| BRC-56 | Wallet Wire Protocol | `WalletClient` communicates with extensions via BRC-56 |

---

## Testing

### Start the Server

```bash
cd backend
npm install
npm run build
npm start
# Running on http://localhost:3002
```

### Test Endpoints

```bash
# Certifier discovery
curl http://localhost:3002/api/info

# Issue certificate (requires a valid BSV identity key)
curl -X POST http://localhost:3002/api/certify \
  -H "Content-Type: application/json" \
  -d '{"identityKey":"03d4e5f6..."}'

# Protected route (will return 401 without AuthFetch)
curl http://localhost:3002/protected/dashboard
```

---

## Summary

This backend guide covered:

- **Server Identity** — Deriving certifier public key from private key
- **Wallet Types** — ProtoWallet for signing, full Wallet for auth middleware
- **Certificate Issuance** — `MasterCertificate.issueCertificateForSubject()` with field encryption
- **Auth Middleware** — `createAuthMiddleware` for mutual authentication
- **Field Decryption** — `VerifiableCertificate.decryptFields()` for server-side verification
- **Protected Routes** — Certificate-gated endpoints with verification tracking
- **Revocation** — Session and verification cleanup

These patterns form the foundation for building self-hosted certification servers on BSV.

---

## Next Steps

- [Frontend Application Guide](./frontend.md) — Build the browser-side interface
- [BSV SDK Documentation](https://docs.bsvblockchain.org/) — Full SDK reference
- [BRC-52: Certificate Creation](https://brc.dev/52) — Certificate standard specification
- [BRC-31: Auth Protocol](https://brc.dev/31) — Authentication protocol details
