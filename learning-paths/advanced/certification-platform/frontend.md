# Certification Platform - Frontend Guide

**Building the Browser-Side Certification Interface with BSV SDK**

This guide covers frontend patterns for implementing certificate acquisition, wallet integration, and authenticated access to protected content using WalletClient and AuthFetch.

> **Complete Code Repository**: [https://github.com/bsv-blockchain-demos/certification-platform](https://github.com/bsv-blockchain-demos/certification-platform)
>
> All code examples in this guide are taken from the working implementation in the `frontend/` directory.

---

## Table of Contents

1. [Setup](#setup)
2. [Wallet Integration](#wallet-integration)
3. [Certifier Discovery](#certifier-discovery)
4. [Certificate Acquisition](#certificate-acquisition)
5. [Certificate Revocation](#certificate-revocation)
6. [Authenticated Dashboard](#authenticated-dashboard)
7. [Complete Flow](#complete-flow)

---

## Setup

### Installation

```bash
npm install @bsv/sdk next react react-dom
npm install -D typescript @types/react @types/node
```

### Environment Configuration

Create `.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:3002
```

> **Note**: The `NEXT_PUBLIC_` prefix exposes this variable to the browser. The backend must be running on the configured port.

### Project Structure

```
frontend/
├── src/
│   ├── app/
│   │   ├── layout.tsx              # Root layout with WalletProvider
│   │   ├── page.tsx                # Certificate issuance page
│   │   └── dashboard/
│   │       └── page.tsx            # Protected dashboard (AuthFetch)
│   ├── context/
│   │   └── WalletContext.tsx        # WalletClient React context
│   ├── components/
│   │   └── Nav.tsx                 # Navigation component
│   └── constants.ts                # API URL config
├── package.json
├── next.config.mjs
├── tailwind.config.js
└── tsconfig.json
```

---

## Wallet Integration

### Pattern 1: WalletClient React Context

WalletClient connects to the user's BSV wallet extension. A React context provides wallet access across the entire application:

> **See full implementation**: `frontend/src/context/WalletContext.tsx`

```typescript
// src/context/WalletContext.tsx

'use client'

import React, { createContext, useContext, useState, useEffect } from 'react'
import { WalletClient } from '@bsv/sdk'

interface WalletContextType {
  wallet: WalletClient | null
  identityKey: string | null
  isInitialized: boolean
  error: string | null
}

const WalletContext = createContext<WalletContextType | undefined>(undefined)

export const WalletProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [wallet, setWallet] = useState<WalletClient | null>(null)
  const [identityKey, setIdentityKey] = useState<string | null>(null)
  const [isInitialized, setIsInitialized] = useState(false)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const initWallet = async () => {
      try {
        const walletClient = new WalletClient()
        const { publicKey } = await walletClient.getPublicKey({ identityKey: true })
        setWallet(walletClient)
        setIdentityKey(publicKey)
        setIsInitialized(true)
      } catch (err) {
        console.error('Failed to initialize wallet:', err)
        setError('Failed to initialize wallet. Please ensure a compatible BSV wallet extension is installed.')
        setIsInitialized(false)
      }
    }

    initWallet()
  }, [])

  return (
    <WalletContext.Provider value={{ wallet, identityKey, isInitialized, error }}>
      {children}
    </WalletContext.Provider>
  )
}

export const useWallet = () => {
  const context = useContext(WalletContext)
  if (context === undefined) {
    throw new Error('useWallet must be used within a WalletProvider')
  }
  return context
}
```

**Key Points:**
- `WalletClient()` auto-discovers the browser wallet extension (no constructor arguments)
- `getPublicKey({ identityKey: true })` returns the user's 33-byte compressed public key
- The identity key is used as the certificate `subject` and for authentication
- No private keys leave the wallet extension — all signing happens inside it

**Mount the Provider** in the root layout:

```typescript
// src/app/layout.tsx

import { WalletProvider } from '@/context/WalletContext'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <WalletProvider>
          <Nav />
          <main>{children}</main>
        </WalletProvider>
      </body>
    </html>
  )
}
```

> **Reference**: [WalletClient Integration](../../beginner/wallet-client-integration/README.md)

---

## Certifier Discovery

### Pattern 2: Fetching Certifier Info

Before acquiring certificates, the frontend must discover the server's identity (certifier public key) and certificate type:

```typescript
interface CertifierInfo {
  certifierPublicKey: string   // Server's identity (33-byte compressed hex)
  certificateType: string      // Base64-encoded type: "Y2VydGlmaWNhdGlvbg=="
}

// Fetch certifier info from backend
const infoRes = await fetch(`${API_BASE_URL}/api/info`)
const info: CertifierInfo = await infoRes.json()
```

**Why Discovery?**
- The certifier public key is needed to filter certificates in the wallet
- The certificate type identifies what kind of certificate to look for
- Both values come from the server's private key and are deterministic

### Checking for Existing Certificates

After discovering the certifier, check if the user already has a certificate:

```typescript
const result = await wallet.listCertificates({
  certifiers: [info.certifierPublicKey],  // Only certs from our server
  types: [info.certificateType],          // Only our certificate type
  limit: 10
})

if (result.certificates && result.certificates.length > 0) {
  // User already has a certificate
  const serialNumber = result.certificates[0].serialNumber
}
```

**`listCertificates` Parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `certifiers` | `string[]` | Filter by certifier public keys |
| `types` | `string[]` | Filter by certificate type identifiers |
| `limit` | `number` | Maximum certificates to return |

> **Reference**: [WalletClient.listCertificates](https://docs.bsvblockchain.org/)

---

## Certificate Acquisition

### Pattern 3: Direct Acquisition Protocol

The core issuance flow: request a certificate from the server, then store it in the wallet using the direct protocol.

> **See full implementation**: `frontend/src/app/page.tsx`

**Step 1: Request Certificate from Server**

```typescript
const certRes = await fetch(`${API_BASE_URL}/api/certify`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ identityKey })  // User's public key
})

const certData = await certRes.json()
```

The server returns a complete, signed, encrypted certificate:

```typescript
// certData structure
{
  type: "Y2VydGlmaWNhdGlvbg==",           // base64('certification')
  serialNumber: "a1b2c3d4...",              // Unique hex identifier
  subject: "03d4e5f6...",                   // User's public key
  certifier: "02a1b2c3...",                 // Server's public key
  fields: {
    certified: "<encrypted>",               // AES-encrypted field value
    timestamp: "<encrypted>"                // AES-encrypted field value
  },
  signature: "304402...",                   // ECDSA signature
  revocationOutpoint: "000...000.0",        // Revocation UTXO pointer
  keyringForSubject: {
    certified: "<encrypted-shared-key>",    // Subject can decrypt this
    timestamp: "<encrypted-shared-key>"     // Subject can decrypt this
  }
}
```

**Step 2: Store in Wallet**

```typescript
await wallet.acquireCertificate({
  type:                certData.type,
  certifier:           certData.certifier,
  acquisitionProtocol: 'direct',            // No external handshake
  fields:              certData.fields,
  serialNumber:        certData.serialNumber,
  revocationOutpoint:  certData.revocationOutpoint,
  signature:           certData.signature,
  keyringRevealer:     'certifier',         // Keyring encrypted by certifier
  keyringForSubject:   certData.keyringForSubject
})
```

**`acquireCertificate` Parameters:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `type` | `"Y2VydGlmaWNhdGlvbg=="` | Certificate type identifier |
| `certifier` | Server's public key | Who signed this certificate |
| `acquisitionProtocol` | `'direct'` | Skip external handshake, store directly |
| `fields` | Encrypted field values | `{ certified: "<enc>", timestamp: "<enc>" }` |
| `serialNumber` | Unique hex string | Certificate instance ID |
| `revocationOutpoint` | `"000...0.0"` | Revocation UTXO pointer |
| `signature` | ECDSA hex | Certifier's signature over the certificate |
| `keyringRevealer` | `'certifier'` | Tells wallet the keyring was encrypted by the certifier |
| `keyringForSubject` | Encrypted keys per field | Lets the subject decrypt field values |

### Why `'direct'` Protocol?

The BSV SDK supports two acquisition protocols:

| Protocol | Flow | When to Use |
|----------|------|-------------|
| `'issuance'` | Client ↔ external certifier URL handshake | Third-party certifier |
| `'direct'` | Client stores pre-signed cert data | Your server IS the certifier |

Since this platform's backend signs the certificate itself, the data is already complete. No external negotiation is needed.

### Why `keyringRevealer: 'certifier'`?

This tells the wallet that the keyring entries were encrypted by the certifier's key. The wallet uses the certifier's public key as the ECDH counterparty to recover the shared secrets and decrypt field values.

### Handling Re-certification

Before acquiring a new certificate, remove any existing ones to prevent duplicates:

```typescript
const handleCertify = async () => {
  // Check for existing certificates
  const existingCerts = await wallet.listCertificates({
    certifiers: [certifierInfo.certifierPublicKey],
    types: [certifierInfo.certificateType],
    limit: 10
  })

  // Remove old certificates
  if (existingCerts.certificates && existingCerts.certificates.length > 0) {
    for (const cert of existingCerts.certificates) {
      await wallet.relinquishCertificate({
        type: certifierInfo.certificateType,
        serialNumber: cert.serialNumber,
        certifier: certifierInfo.certifierPublicKey
      })
    }
  }

  // Request new certificate from server
  const certRes = await fetch(`${API_BASE_URL}/api/certify`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ identityKey })
  })
  const certData = await certRes.json()

  // Store in wallet
  await wallet.acquireCertificate({
    type:                certData.type,
    certifier:           certData.certifier,
    acquisitionProtocol: 'direct',
    fields:              certData.fields,
    serialNumber:        certData.serialNumber,
    revocationOutpoint:  certData.revocationOutpoint,
    signature:           certData.signature,
    keyringRevealer:     'certifier',
    keyringForSubject:   certData.keyringForSubject
  })
}
```

---

## Certificate Revocation

### Pattern 4: Removing Certificates

Users can revoke their certificate, removing it from the wallet:

```typescript
await wallet.relinquishCertificate({
  type: certifierInfo.certificateType,
  serialNumber: certificateSerialNumber,
  certifier: certifierInfo.certifierPublicKey
})
```

The three-part key `(type, serialNumber, certifier)` uniquely identifies a certificate in the wallet.

**Complete Revocation** also notifies the server to clean up sessions:

```typescript
const handleRelinquish = async () => {
  // 1. Remove from wallet
  await wallet.relinquishCertificate({
    type: certifierInfo.certificateType,
    serialNumber: certificateSerialNumber,
    certifier: certifierInfo.certifierPublicKey
  })

  // 2. Update UI state
  setHasCertificate(false)
  setCertificateSerialNumber(null)
}
```

When the user subsequently tries to access the protected dashboard, AuthFetch will fail the auth challenge (no certificate to present), and the server-side session will eventually be cleaned up.

---

## Authenticated Dashboard

### Pattern 5: AuthFetch for Gated Content

`AuthFetch` from `@bsv/sdk` wraps the standard `fetch()` API and handles the mutual authentication protocol transparently:

> **See full implementation**: `frontend/src/app/dashboard/page.tsx`

```typescript
import { AuthFetch } from '@bsv/sdk'

const authFetch = new AuthFetch(wallet)  // wallet = WalletClient instance

const response = await authFetch.fetch(
  `${API_BASE_URL}/protected/dashboard`,
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' }
  }
)

if (response.ok) {
  const data = await response.json()
  // Access granted — display protected content
} else {
  // Access denied — no valid certificate
}
```

**What AuthFetch Does Internally:**

```
authFetch.fetch(url)
       │
       ├── 1. Send request (no session yet)
       │
       ├── 2. Server returns 401 + challenge (nonce, server identity)
       │
       ├── 3. AuthFetch automatically:
       │   ├── Gets identity key from wallet
       │   ├── Signs the nonce with wallet
       │   ├── Calls wallet.listCertificates() to find matching certs
       │   ├── Includes certificates + keyrings in response
       │   └── Retries the request with auth payload
       │
       ├── 4. Server verifies signature, decrypts cert fields
       │      → creates session, returns 200
       │
       └── 5. Subsequent requests use cached session token
```

**Key Difference from Plain `fetch()`:**

| Feature | `fetch()` | `AuthFetch.fetch()` |
|---------|-----------|---------------------|
| Authentication | None | Automatic mutual auth |
| Certificate presentation | Manual | Automatic from wallet |
| Session management | Manual | Automatic caching |
| Use for public endpoints | Yes | Unnecessary overhead |
| Use for protected endpoints | Cannot work | Required |

Use plain `fetch()` for public endpoints (`/api/info`, `/api/certify`) and `AuthFetch` for protected endpoints (`/protected/dashboard`).

### Dashboard Component

```typescript
'use client'

import { useState, useEffect } from 'react'
import { useWallet } from '@/context/WalletContext'
import { API_BASE_URL } from '@/constants'
import { AuthFetch } from '@bsv/sdk'

interface DashboardData {
  status: string
  accessLevel: string
  certifiedAt: string
}

export default function DashboardPage() {
  const { wallet, isInitialized } = useWallet()
  const [isVerified, setIsVerified] = useState(false)
  const [dashboardData, setDashboardData] = useState<DashboardData | null>(null)
  const [isChecking, setIsChecking] = useState(true)

  useEffect(() => {
    if (!wallet || !isInitialized) return

    const checkAccess = async () => {
      try {
        const authFetch = new AuthFetch(wallet)
        const response = await authFetch.fetch(
          `${API_BASE_URL}/protected/dashboard`,
          {
            method: 'GET',
            headers: { 'Content-Type': 'application/json' }
          }
        )

        if (response.ok) {
          const data = await response.json()
          setDashboardData(data.data)
          setIsVerified(true)
        }
      } catch (err) {
        console.error('Failed to access dashboard:', err)
      } finally {
        setIsChecking(false)
      }
    }

    checkAccess()
  }, [wallet, isInitialized])

  if (isChecking) return <p>Verifying certificate...</p>
  if (!isVerified) return <p>Certificate required. Get certified first.</p>

  return (
    <div>
      <h1>Dashboard</h1>
      <p>Status: {dashboardData?.status}</p>
      <p>Access Level: {dashboardData?.accessLevel}</p>
      <p>Certified At: {dashboardData?.certifiedAt}</p>
    </div>
  )
}
```

---

## Complete Flow

### End-to-End Certificate Acquisition and Access

```typescript
// ============================================
// PHASE 1: DISCOVERY
// ============================================

// Wallet connects automatically via WalletProvider
const { wallet, identityKey } = useWallet()

// Fetch certifier info
const infoRes = await fetch(`${API_BASE_URL}/api/info`)
const info = await infoRes.json()
// → { certifierPublicKey: "02a1b2c3...", certificateType: "Y2VydGlm..." }

// Check for existing certificate
const existing = await wallet.listCertificates({
  certifiers: [info.certifierPublicKey],
  types: [info.certificateType],
  limit: 10
})

if (existing.certificates.length > 0) {
  // Already certified — can access dashboard
}

// ============================================
// PHASE 2: ISSUANCE
// ============================================

// Request certificate from server
const certRes = await fetch(`${API_BASE_URL}/api/certify`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ identityKey })
})
const certData = await certRes.json()
// → { type, serialNumber, fields(encrypted), signature, keyringForSubject, ... }

// Store in wallet via direct protocol
await wallet.acquireCertificate({
  type:                certData.type,
  certifier:           certData.certifier,
  acquisitionProtocol: 'direct',
  fields:              certData.fields,
  serialNumber:        certData.serialNumber,
  revocationOutpoint:  certData.revocationOutpoint,
  signature:           certData.signature,
  keyringRevealer:     'certifier',
  keyringForSubject:   certData.keyringForSubject
})

// ============================================
// PHASE 3: GATED ACCESS
// ============================================

// Access protected content with AuthFetch
const authFetch = new AuthFetch(wallet)
const response = await authFetch.fetch(`${API_BASE_URL}/protected/dashboard`)
// → AuthFetch handles: 401 challenge → sign nonce → present cert → session
const data = await response.json()
// → { success: true, data: { status, accessLevel, certifiedAt } }

// ============================================
// PHASE 4: REVOCATION
// ============================================

// Remove certificate from wallet
await wallet.relinquishCertificate({
  type: info.certificateType,
  serialNumber: certData.serialNumber,
  certifier: info.certifierPublicKey
})
// → Certificate removed, dashboard access revoked
```

---

## Important Concepts

### WalletClient Methods Used

| Method | Purpose | When Called |
|--------|---------|------------|
| `getPublicKey({ identityKey: true })` | Get user's identity | App initialization |
| `listCertificates({ certifiers, types })` | Check existing certs | Page mount |
| `acquireCertificate({ ... })` | Store new certificate | After server issuance |
| `relinquishCertificate({ ... })` | Remove certificate | User revocation |

### AuthFetch vs Plain Fetch

```
Public endpoints (no auth needed):
  GET  /api/info      → use fetch()
  POST /api/certify   → use fetch()

Protected endpoints (cert required):
  GET  /protected/dashboard → use AuthFetch.fetch()
  POST /relinquish          → use AuthFetch.fetch()
```

### Certificate Uniqueness

A certificate is uniquely identified by three fields:
- **type** — the base64-encoded certificate category
- **serialNumber** — unique instance identifier
- **certifier** — public key of the signing authority

### Error Handling

Always handle wallet and network errors gracefully:

```typescript
try {
  await wallet.acquireCertificate({ ... })
} catch (err: unknown) {
  const message = err instanceof Error ? err.message : 'Failed to acquire certificate'
  setError(message)
}
```

Common errors:
- **"Wallet not connected"** — WalletClient not initialized
- **"Certificate already exists"** — duplicate serial number
- **"Invalid signature"** — certificate data was tampered with
- **"Network error"** — backend not reachable

---

## Summary

This frontend guide covered:

- **WalletClient Context** — React provider for wallet access across components
- **Certifier Discovery** — Fetching server identity and certificate type
- **Certificate Acquisition** — Direct protocol for storing pre-signed certificates
- **Certificate Revocation** — Removing certificates from the wallet
- **AuthFetch** — Transparent mutual authentication for protected routes
- **Complete Integration** — End-to-end flow from discovery to gated access

These patterns form the foundation for building certificate-gated frontends on BSV.

---

## Next Steps

- [Backend Server Guide](./backend.md) — Learn how the server issues and verifies certificates
- [WalletClient Integration](../../beginner/wallet-client-integration/README.md) — WalletClient fundamentals
- [BSV SDK Documentation](https://docs.bsvblockchain.org/) — Full SDK reference
- [BRC-56: Wallet Wire Protocol](https://brc.dev/56) — How WalletClient communicates with extensions
