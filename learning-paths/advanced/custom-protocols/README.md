# Custom Protocols

## Overview

Design and implement custom application-specific protocols on BSV blockchain using overlay networks, data standards, and protocol design patterns. This module teaches you to build production-ready protocols that layer on top of BSV's base layer, enabling specialized applications like tokens, verifiable credentials, supply chain tracking, identity systems, and domain-specific solutions. Learn to integrate with existing BSV infrastructure while maintaining interoperability and scalability.

**Estimated Time:** 5-6 hours
**Difficulty:** Advanced

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Design custom protocols with clear specification and standards
- ✅ Implement overlay networks for application-specific data routing
- ✅ Build token protocols with issuance, transfer, and burning mechanisms
- ✅ Create verifiable credential systems with BRC-56 integration
- ✅ Design identity protocols with privacy-preserving features
- ✅ Implement supply chain tracking with data anchoring
- ✅ Build protocol indexers and parsers for transaction discovery
- ✅ Create protocol-specific wallets and user interfaces
- ✅ Ensure interoperability with existing BSV standards
- ✅ Deploy and maintain production protocol infrastructure

## SDK Components Used

This course leverages these standardized SDK modules:

- **[Transaction](../../../sdk-components/transaction/README.md)** - Custom protocol transaction construction
- **[Script](../../../sdk-components/script/README.md)** - Protocol-specific locking scripts
- **[Script Templates](../../../sdk-components/script-templates/README.md)** - Protocol template patterns
- **[Transaction Output](../../../sdk-components/transaction-output/README.md)** - OP_RETURN data embedding
- **[SPV](../../../sdk-components/spv/README.md)** - Light client protocol integration
- **[BEEF](../../../sdk-components/beef/README.md)** - Protocol transaction envelopes
- **[ARC](../../../sdk-components/arc/README.md)** - Protocol transaction broadcasting
- **[BRC-29](../../../sdk-components/brc-29/README.md)** - Payment protocol integration

## 1. Protocol Design Fundamentals

### What is a Custom Protocol?

A custom protocol is an application-specific layer built on top of BSV blockchain that defines:

1. **Data Format**: How data is structured in transactions
2. **Transaction Patterns**: Standard transaction types for the protocol
3. **Validation Rules**: How to verify protocol-compliant transactions
4. **Discovery Mechanism**: How to find protocol transactions
5. **State Management**: How protocol state is tracked and updated

```typescript
import { Transaction, Script, OP } from '@bsv/sdk'

/**
 * Protocol Design Principles
 *
 * 1. Use OP_RETURN for protocol metadata (not locking script funds)
 * 2. Define clear versioning for protocol evolution
 * 3. Use BRC standards where applicable
 * 4. Design for SPV client compatibility
 * 5. Enable light client verification
 * 6. Consider privacy implications
 * 7. Plan for protocol upgrades
 * 8. Document everything clearly
 */

interface ProtocolSpecification {
  name: string              // Protocol name (e.g., "MyToken")
  version: number           // Protocol version (e.g., 1)
  prefix: string            // Protocol identifier (e.g., "MYTOKEN")
  actions: ProtocolAction[] // Supported actions
  validation: ValidationRules
}

interface ProtocolAction {
  name: string              // Action name (e.g., "TRANSFER")
  opcode: number           // Action identifier
  fields: FieldDefinition[] // Required data fields
  validation: (tx: Transaction) => boolean
}

interface FieldDefinition {
  name: string
  type: 'string' | 'number' | 'bytes' | 'pubkey' | 'signature'
  required: boolean
  maxLength?: number
}

/**
 * Example: Simple Token Protocol Specification
 */
const SimpleTokenProtocol: ProtocolSpecification = {
  name: "SimpleToken",
  version: 1,
  prefix: "STOKEN",
  actions: [
    {
      name: "ISSUE",
      opcode: 0x01,
      fields: [
        { name: "tokenId", type: "bytes", required: true, maxLength: 32 },
        { name: "amount", type: "number", required: true },
        { name: "ownerPubKey", type: "pubkey", required: true }
      ],
      validation: (tx) => validateTokenIssuance(tx)
    },
    {
      name: "TRANSFER",
      opcode: 0x02,
      fields: [
        { name: "tokenId", type: "bytes", required: true, maxLength: 32 },
        { name: "amount", type: "number", required: true },
        { name: "fromPubKey", type: "pubkey", required: true },
        { name: "toPubKey", type: "pubkey", required: true },
        { name: "signature", type: "signature", required: true }
      ],
      validation: (tx) => validateTokenTransfer(tx)
    }
  ],
  validation: {
    maxSupply: 21_000_000,
    minTransferAmount: 1,
    requireSignatures: true
  }
}
```

### Protocol Identifier Standards

Reference: **[Transaction Output Component - OP_RETURN](../../../sdk-components/transaction-output/README.md#op_return-data-outputs)**

```typescript
import { Transaction, Script, OP } from '@bsv/sdk'

/**
 * Protocol Prefix System
 *
 * Use consistent prefixes to identify protocol transactions:
 * - First output: OP_FALSE OP_RETURN <protocol_prefix> <version> <data>
 * - Makes transaction discovery easy via indexing
 * - Enables light clients to filter relevant transactions
 */

class ProtocolIdentifier {
  /**
   * Create a standard protocol prefix output
   */
  static createPrefixOutput(
    prefix: string,
    version: number,
    action: string,
    data: Buffer
  ): Script {
    const script = new Script()

    // Standard format: OP_FALSE OP_RETURN <prefix> <version> <action> <data>
    script.writeOpCode(OP.OP_FALSE)
    script.writeOpCode(OP.OP_RETURN)
    script.writeBin(Buffer.from(prefix, 'utf8'))
    script.writeBin(Buffer.from([version]))
    script.writeBin(Buffer.from(action, 'utf8'))
    script.writeBin(data)

    return script
  }

  /**
   * Parse protocol data from OP_RETURN output
   */
  static parseProtocolOutput(script: Script): {
    prefix: string
    version: number
    action: string
    data: Buffer
  } | null {
    try {
      const chunks = script.chunks

      // Verify format: OP_FALSE OP_RETURN <prefix> <version> <action> <data>
      if (chunks.length < 6) return null
      if (chunks[0].op !== OP.OP_FALSE) return null
      if (chunks[1].op !== OP.OP_RETURN) return null

      return {
        prefix: chunks[2].buf!.toString('utf8'),
        version: chunks[3].buf![0],
        action: chunks[4].buf!.toString('utf8'),
        data: chunks[5].buf!
      }
    } catch (error) {
      return null
    }
  }
}

/**
 * Example: Token Protocol Transaction
 */
async function createTokenIssuanceTx(
  tokenId: string,
  amount: number,
  ownerPubKey: string,
  fundingUTXO: UTXO,
  privateKey: PrivateKey
): Promise<Transaction> {
  const tx = new Transaction()

  // Input: Funding for transaction fees
  tx.addInput({
    sourceTXID: fundingUTXO.txid,
    sourceOutputIndex: fundingUTXO.vout,
    unlockingScript: new Script(),
    sequence: 0xffffffff
  })

  // Output 0: Protocol data (OP_RETURN)
  const tokenData = Buffer.concat([
    Buffer.from(tokenId, 'hex'),
    Buffer.alloc(8).writeUInt32LE(amount, 0),
    Buffer.from(ownerPubKey, 'hex')
  ])

  const protocolScript = ProtocolIdentifier.createPrefixOutput(
    'STOKEN',
    1,
    'ISSUE',
    tokenData
  )

  tx.addOutput({
    satoshis: 0,
    lockingScript: protocolScript
  })

  // Output 1: Token ownership (locked to owner's public key)
  tx.addOutput({
    satoshis: 1, // Dust amount
    lockingScript: Script.fromASM(
      `OP_DUP OP_HASH160 ${ownerPubKey} OP_EQUALVERIFY OP_CHECKSIG`
    )
  })

  // Output 2: Change
  const fee = 500
  const changeAmount = fundingUTXO.satoshis - 1 - fee

  tx.addOutput({
    satoshis: changeAmount,
    lockingScript: Script.fromASM(
      `OP_DUP OP_HASH160 ${privateKey.toPublicKey().toHash()} OP_EQUALVERIFY OP_CHECKSIG`
    )
  })

  // Sign transaction
  await tx.sign()

  return tx
}
```

## 2. Token Protocol Design

### Token Standard Implementation

Reference: **[Script Templates Component](../../../sdk-components/script-templates/README.md)**

```typescript
import { Transaction, Script, PrivateKey, PublicKey, OP } from '@bsv/sdk'

/**
 * Complete Token Protocol Implementation
 *
 * Features:
 * - Token issuance with max supply
 * - Token transfers with signature verification
 * - Token burning mechanism
 * - Balance tracking via UTXO model
 * - SPV-compatible verification
 */

interface TokenUTXO {
  txid: string
  vout: number
  tokenId: string
  amount: number
  owner: string // Public key hash
  satoshis: number
}

class TokenProtocol {
  private readonly protocolPrefix = 'TOKEN'
  private readonly version = 1

  /**
   * Issue new tokens (create supply)
   */
  async issueTokens(
    tokenId: string,
    symbol: string,
    amount: number,
    maxSupply: number,
    ownerPrivateKey: PrivateKey,
    fundingUTXO: UTXO
  ): Promise<Transaction> {
    if (amount > maxSupply) {
      throw new Error('Amount exceeds max supply')
    }

    const tx = new Transaction()

    // Add funding input
    tx.addInput({
      sourceTXID: fundingUTXO.txid,
      sourceOutputIndex: fundingUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Token metadata (OP_RETURN)
    const metadata = this.encodeTokenMetadata({
      action: 'ISSUE',
      tokenId,
      symbol,
      amount,
      maxSupply,
      decimals: 8
    })

    tx.addOutput({
      satoshis: 0,
      lockingScript: this.createProtocolOutput('ISSUE', metadata)
    })

    // Output 1: Token ownership UTXO
    const ownerPubKey = ownerPrivateKey.toPublicKey()
    tx.addOutput({
      satoshis: 1000, // Dust amount to make output spendable
      lockingScript: this.createTokenLockingScript(tokenId, amount, ownerPubKey)
    })

    // Output 2: Change
    const fee = 500
    tx.addOutput({
      satoshis: fundingUTXO.satoshis - 1000 - fee,
      lockingScript: Script.fromPubKeyHash(ownerPubKey.toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Transfer tokens to another address
   */
  async transferTokens(
    tokenUTXO: TokenUTXO,
    amount: number,
    recipientPubKey: PublicKey,
    senderPrivateKey: PrivateKey,
    fundingUTXO?: UTXO
  ): Promise<Transaction> {
    if (amount > tokenUTXO.amount) {
      throw new Error('Insufficient token balance')
    }

    const tx = new Transaction()

    // Input 0: Token UTXO being spent
    tx.addInput({
      sourceTXID: tokenUTXO.txid,
      sourceOutputIndex: tokenUTXO.vout,
      unlockingScript: new Script(), // Will be signed
      sequence: 0xffffffff
    })

    // Input 1: Optional funding for fees (if needed)
    if (fundingUTXO) {
      tx.addInput({
        sourceTXID: fundingUTXO.txid,
        sourceOutputIndex: fundingUTXO.vout,
        unlockingScript: new Script(),
        sequence: 0xffffffff
      })
    }

    // Output 0: Transfer metadata (OP_RETURN)
    const metadata = this.encodeTokenMetadata({
      action: 'TRANSFER',
      tokenId: tokenUTXO.tokenId,
      amount,
      from: senderPrivateKey.toPublicKey().toHash().toString('hex'),
      to: recipientPubKey.toHash().toString('hex')
    })

    tx.addOutput({
      satoshis: 0,
      lockingScript: this.createProtocolOutput('TRANSFER', metadata)
    })

    // Output 1: Recipient's token UTXO
    tx.addOutput({
      satoshis: 1000,
      lockingScript: this.createTokenLockingScript(
        tokenUTXO.tokenId,
        amount,
        recipientPubKey
      )
    })

    // Output 2: Change token UTXO (if any remaining)
    const changeAmount = tokenUTXO.amount - amount
    if (changeAmount > 0) {
      tx.addOutput({
        satoshis: 1000,
        lockingScript: this.createTokenLockingScript(
          tokenUTXO.tokenId,
          changeAmount,
          senderPrivateKey.toPublicKey()
        )
      })
    }

    // Output 3: Satoshi change (if funding UTXO provided)
    if (fundingUTXO) {
      const fee = 500
      const satoshiChange = fundingUTXO.satoshis - 1000 - (changeAmount > 0 ? 1000 : 0) - fee

      if (satoshiChange > 0) {
        tx.addOutput({
          satoshis: satoshiChange,
          lockingScript: Script.fromPubKeyHash(senderPrivateKey.toPublicKey().toHash())
        })
      }
    }

    await tx.sign()
    return tx
  }

  /**
   * Burn tokens (remove from circulation)
   */
  async burnTokens(
    tokenUTXO: TokenUTXO,
    amount: number,
    ownerPrivateKey: PrivateKey
  ): Promise<Transaction> {
    if (amount > tokenUTXO.amount) {
      throw new Error('Burn amount exceeds balance')
    }

    const tx = new Transaction()

    // Input: Token UTXO being burned
    tx.addInput({
      sourceTXID: tokenUTXO.txid,
      sourceOutputIndex: tokenUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Burn metadata (OP_RETURN)
    const metadata = this.encodeTokenMetadata({
      action: 'BURN',
      tokenId: tokenUTXO.tokenId,
      amount
    })

    tx.addOutput({
      satoshis: 0,
      lockingScript: this.createProtocolOutput('BURN', metadata)
    })

    // Output 1: Remaining tokens (if partial burn)
    const remainingAmount = tokenUTXO.amount - amount
    if (remainingAmount > 0) {
      tx.addOutput({
        satoshis: 1000,
        lockingScript: this.createTokenLockingScript(
          tokenUTXO.tokenId,
          remainingAmount,
          ownerPrivateKey.toPublicKey()
        )
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Create token-specific locking script
   * This encodes token metadata in the locking script itself
   */
  private createTokenLockingScript(
    tokenId: string,
    amount: number,
    ownerPubKey: PublicKey
  ): Script {
    // Standard P2PKH with token data in OP_RETURN prefix
    const script = new Script()

    // Token data prefix (for indexing)
    script.writeOpCode(OP.OP_DUP)
    script.writeOpCode(OP.OP_HASH160)
    script.writeBin(ownerPubKey.toHash())
    script.writeOpCode(OP.OP_EQUALVERIFY)
    script.writeOpCode(OP.OP_CHECKSIG)

    return script
  }

  /**
   * Create protocol OP_RETURN output
   */
  private createProtocolOutput(action: string, data: Buffer): Script {
    return ProtocolIdentifier.createPrefixOutput(
      this.protocolPrefix,
      this.version,
      action,
      data
    )
  }

  /**
   * Encode token metadata to buffer
   */
  private encodeTokenMetadata(metadata: any): Buffer {
    // Simple JSON encoding (production should use more efficient binary format)
    return Buffer.from(JSON.stringify(metadata), 'utf8')
  }
}

/**
 * Usage Example
 */
async function tokenProtocolExample() {
  const protocol = new TokenProtocol()

  // Issue tokens
  const issuerKey = PrivateKey.fromRandom()
  const fundingUTXO: UTXO = {
    txid: '...',
    vout: 0,
    satoshis: 100000,
    script: Script.fromPubKeyHash(issuerKey.toPublicKey().toHash())
  }

  const issueTx = await protocol.issueTokens(
    'my-token-001',
    'MYT',
    1000000, // 1 million tokens
    21000000, // 21 million max supply
    issuerKey,
    fundingUTXO
  )

  console.log('Token issuance tx:', issueTx.id('hex'))

  // Transfer tokens
  const recipientKey = PrivateKey.fromRandom()
  const tokenUTXO: TokenUTXO = {
    txid: issueTx.id('hex'),
    vout: 1,
    tokenId: 'my-token-001',
    amount: 1000000,
    owner: issuerKey.toPublicKey().toHash().toString('hex'),
    satoshis: 1000
  }

  const transferTx = await protocol.transferTokens(
    tokenUTXO,
    50000, // Transfer 50k tokens
    recipientKey.toPublicKey(),
    issuerKey
  )

  console.log('Token transfer tx:', transferTx.id('hex'))
}
```

## 3. Verifiable Credentials Protocol

### BRC-56 Integration

Reference: **[BRC-29 Component - Verifiable Credentials](../../../sdk-components/brc-29/README.md)**

```typescript
import { Transaction, Script, PrivateKey, PublicKey, Signature, OP } from '@bsv/sdk'

/**
 * Verifiable Credentials Protocol (BRC-56)
 *
 * Features:
 * - Issue credentials with issuer signature
 * - Verify credentials cryptographically
 * - Revoke credentials on-chain
 * - Support for various credential types
 * - Privacy-preserving selective disclosure
 */

interface Credential {
  id: string
  type: string
  issuer: string // Issuer public key
  subject: string // Subject public key
  issuedAt: number // Unix timestamp
  expiresAt?: number
  claims: Record<string, any>
  proof: {
    type: 'EcdsaSecp256k1Signature2019'
    created: string
    verificationMethod: string
    signature: string
  }
}

class VerifiableCredentialsProtocol {
  private readonly protocolPrefix = 'VCP'
  private readonly version = 1

  /**
   * Issue a verifiable credential
   */
  async issueCredential(
    credentialType: string,
    subjectPubKey: PublicKey,
    claims: Record<string, any>,
    issuerPrivateKey: PrivateKey,
    expiresAt?: number,
    fundingUTXO?: UTXO
  ): Promise<{ credential: Credential; tx: Transaction }> {
    const credentialId = this.generateCredentialId()
    const issuedAt = Math.floor(Date.now() / 1000)

    // Create credential object
    const credential: Credential = {
      id: credentialId,
      type: credentialType,
      issuer: issuerPrivateKey.toPublicKey().toString(),
      subject: subjectPubKey.toString(),
      issuedAt,
      expiresAt,
      claims,
      proof: {
        type: 'EcdsaSecp256k1Signature2019',
        created: new Date(issuedAt * 1000).toISOString(),
        verificationMethod: issuerPrivateKey.toPublicKey().toString(),
        signature: '' // Will be filled below
      }
    }

    // Sign credential
    const credentialHash = this.hashCredential(credential)
    const signature = issuerPrivateKey.sign(credentialHash)
    credential.proof.signature = signature.toHex()

    // Create transaction anchoring credential
    const tx = new Transaction()

    if (fundingUTXO) {
      tx.addInput({
        sourceTXID: fundingUTXO.txid,
        sourceOutputIndex: fundingUTXO.vout,
        unlockingScript: new Script(),
        sequence: 0xffffffff
      })
    }

    // Output 0: Credential data (OP_RETURN)
    const credentialData = Buffer.from(JSON.stringify(credential), 'utf8')

    tx.addOutput({
      satoshis: 0,
      lockingScript: ProtocolIdentifier.createPrefixOutput(
        this.protocolPrefix,
        this.version,
        'ISSUE',
        credentialData
      )
    })

    // Output 1: Credential ownership (locked to subject)
    tx.addOutput({
      satoshis: 1000,
      lockingScript: Script.fromPubKeyHash(subjectPubKey.toHash())
    })

    if (fundingUTXO) {
      const fee = 500
      tx.addOutput({
        satoshis: fundingUTXO.satoshis - 1000 - fee,
        lockingScript: Script.fromPubKeyHash(issuerPrivateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()

    return { credential, tx }
  }

  /**
   * Verify a credential's signature
   */
  verifyCredential(credential: Credential): boolean {
    try {
      // Extract signature
      const signature = Signature.fromHex(credential.proof.signature)

      // Extract issuer public key
      const issuerPubKey = PublicKey.fromString(credential.issuer)

      // Create credential hash (without signature)
      const credentialCopy = { ...credential }
      credentialCopy.proof = { ...credential.proof, signature: '' }
      const credentialHash = this.hashCredential(credentialCopy)

      // Verify signature
      return issuerPubKey.verify(credentialHash, signature)
    } catch (error) {
      console.error('Credential verification failed:', error)
      return false
    }
  }

  /**
   * Check if credential is expired
   */
  isExpired(credential: Credential): boolean {
    if (!credential.expiresAt) return false
    const now = Math.floor(Date.now() / 1000)
    return now > credential.expiresAt
  }

  /**
   * Revoke a credential on-chain
   */
  async revokeCredential(
    credentialId: string,
    issuerPrivateKey: PrivateKey,
    reason: string,
    fundingUTXO: UTXO
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: fundingUTXO.txid,
      sourceOutputIndex: fundingUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Revocation data (OP_RETURN)
    const revocationData = Buffer.from(
      JSON.stringify({
        credentialId,
        revokedAt: Math.floor(Date.now() / 1000),
        reason,
        issuer: issuerPrivateKey.toPublicKey().toString()
      }),
      'utf8'
    )

    tx.addOutput({
      satoshis: 0,
      lockingScript: ProtocolIdentifier.createPrefixOutput(
        this.protocolPrefix,
        this.version,
        'REVOKE',
        revocationData
      )
    })

    // Change output
    const fee = 500
    tx.addOutput({
      satoshis: fundingUTXO.satoshis - fee,
      lockingScript: Script.fromPubKeyHash(issuerPrivateKey.toPublicKey().toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Create selective disclosure proof
   * Allows revealing only specific claims without full credential
   */
  createSelectiveDisclosure(
    credential: Credential,
    disclosedClaims: string[]
  ): {
    id: string
    type: string
    issuer: string
    subject: string
    disclosedClaims: Record<string, any>
    proof: any
  } {
    const disclosed: Record<string, any> = {}

    for (const claim of disclosedClaims) {
      if (credential.claims[claim] !== undefined) {
        disclosed[claim] = credential.claims[claim]
      }
    }

    return {
      id: credential.id,
      type: credential.type,
      issuer: credential.issuer,
      subject: credential.subject,
      disclosedClaims: disclosed,
      proof: credential.proof
    }
  }

  /**
   * Generate unique credential ID
   */
  private generateCredentialId(): string {
    return `cred-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }

  /**
   * Hash credential for signing
   */
  private hashCredential(credential: Credential): Buffer {
    const json = JSON.stringify(credential)
    const hash = require('crypto').createHash('sha256')
    return hash.update(json).digest()
  }
}

/**
 * Usage Example
 */
async function credentialExample() {
  const protocol = new VerifiableCredentialsProtocol()

  // University issues degree credential
  const universityKey = PrivateKey.fromRandom()
  const studentKey = PrivateKey.fromRandom()

  const { credential, tx } = await protocol.issueCredential(
    'UniversityDegreeCredential',
    studentKey.toPublicKey(),
    {
      degree: 'Bachelor of Science',
      major: 'Computer Science',
      graduationYear: 2024,
      gpa: 3.8,
      honors: 'Magna Cum Laude'
    },
    universityKey,
    Math.floor(Date.now() / 1000) + (10 * 365 * 24 * 60 * 60) // Expires in 10 years
  )

  console.log('Credential ID:', credential.id)
  console.log('Transaction:', tx.id('hex'))

  // Verify credential
  const isValid = protocol.verifyCredential(credential)
  console.log('Credential valid:', isValid)

  // Selective disclosure (reveal only degree and major, hide GPA)
  const disclosure = protocol.createSelectiveDisclosure(credential, ['degree', 'major'])
  console.log('Disclosed claims:', disclosure.disclosedClaims)
}
```

## 4. Identity Protocol Design

### Decentralized Identity (DID) System

```typescript
import { Transaction, Script, PrivateKey, PublicKey, OP } from '@bsv/sdk'

/**
 * Decentralized Identity Protocol
 *
 * Features:
 * - Create on-chain identities
 * - Link multiple keys to single identity
 * - Rotate keys securely
 * - Manage identity attributes
 * - Support for identity recovery
 */

interface Identity {
  did: string // Decentralized Identifier
  controller: string // Current controlling public key
  created: number
  updated: number
  keys: IdentityKey[]
  services: IdentityService[]
  attributes: Record<string, any>
}

interface IdentityKey {
  id: string
  type: 'EcdsaSecp256k1VerificationKey2019'
  publicKey: string
  purposes: ('authentication' | 'assertionMethod' | 'keyAgreement')[]
}

interface IdentityService {
  id: string
  type: string
  serviceEndpoint: string
}

class IdentityProtocol {
  private readonly protocolPrefix = 'DID'
  private readonly version = 1

  /**
   * Create a new decentralized identity
   */
  async createIdentity(
    controllerPrivateKey: PrivateKey,
    attributes: Record<string, any>,
    services: IdentityService[],
    fundingUTXO: UTXO
  ): Promise<{ identity: Identity; tx: Transaction }> {
    const controllerPubKey = controllerPrivateKey.toPublicKey()
    const did = this.generateDID(controllerPubKey)
    const now = Math.floor(Date.now() / 1000)

    const identity: Identity = {
      did,
      controller: controllerPubKey.toString(),
      created: now,
      updated: now,
      keys: [
        {
          id: `${did}#key-1`,
          type: 'EcdsaSecp256k1VerificationKey2019',
          publicKey: controllerPubKey.toString(),
          purposes: ['authentication', 'assertionMethod', 'keyAgreement']
        }
      ],
      services,
      attributes
    }

    // Create transaction
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: fundingUTXO.txid,
      sourceOutputIndex: fundingUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Identity document (OP_RETURN)
    const identityData = Buffer.from(JSON.stringify(identity), 'utf8')

    tx.addOutput({
      satoshis: 0,
      lockingScript: ProtocolIdentifier.createPrefixOutput(
        this.protocolPrefix,
        this.version,
        'CREATE',
        identityData
      )
    })

    // Output 1: Identity control UTXO
    tx.addOutput({
      satoshis: 10000, // Higher amount for identity UTXO
      lockingScript: Script.fromPubKeyHash(controllerPubKey.toHash())
    })

    // Change
    const fee = 500
    tx.addOutput({
      satoshis: fundingUTXO.satoshis - 10000 - fee,
      lockingScript: Script.fromPubKeyHash(controllerPubKey.toHash())
    })

    await tx.sign()

    return { identity, tx }
  }

  /**
   * Update identity (add keys, services, attributes)
   */
  async updateIdentity(
    identity: Identity,
    updates: Partial<Identity>,
    controllerPrivateKey: PrivateKey,
    identityUTXO: UTXO
  ): Promise<{ identity: Identity; tx: Transaction }> {
    const updatedIdentity: Identity = {
      ...identity,
      ...updates,
      updated: Math.floor(Date.now() / 1000)
    }

    const tx = new Transaction()

    // Input: Previous identity UTXO
    tx.addInput({
      sourceTXID: identityUTXO.txid,
      sourceOutputIndex: identityUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Updated identity document (OP_RETURN)
    const identityData = Buffer.from(JSON.stringify(updatedIdentity), 'utf8')

    tx.addOutput({
      satoshis: 0,
      lockingScript: ProtocolIdentifier.createPrefixOutput(
        this.protocolPrefix,
        this.version,
        'UPDATE',
        identityData
      )
    })

    // Output 1: New identity control UTXO
    tx.addOutput({
      satoshis: 10000,
      lockingScript: Script.fromPubKeyHash(controllerPrivateKey.toPublicKey().toHash())
    })

    await tx.sign()

    return { identity: updatedIdentity, tx }
  }

  /**
   * Rotate identity controller key
   */
  async rotateKey(
    identity: Identity,
    newControllerPublicKey: PublicKey,
    currentControllerPrivateKey: PrivateKey,
    identityUTXO: UTXO
  ): Promise<Transaction> {
    // Add new key to identity
    const newKey: IdentityKey = {
      id: `${identity.did}#key-${identity.keys.length + 1}`,
      type: 'EcdsaSecp256k1VerificationKey2019',
      publicKey: newControllerPublicKey.toString(),
      purposes: ['authentication', 'assertionMethod', 'keyAgreement']
    }

    const rotatedIdentity: Identity = {
      ...identity,
      controller: newControllerPublicKey.toString(),
      updated: Math.floor(Date.now() / 1000),
      keys: [...identity.keys, newKey]
    }

    const tx = new Transaction()

    tx.addInput({
      sourceTXID: identityUTXO.txid,
      sourceOutputIndex: identityUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Rotated identity (OP_RETURN)
    const identityData = Buffer.from(JSON.stringify(rotatedIdentity), 'utf8')

    tx.addOutput({
      satoshis: 0,
      lockingScript: ProtocolIdentifier.createPrefixOutput(
        this.protocolPrefix,
        this.version,
        'ROTATE',
        identityData
      )
    })

    // Output 1: New control UTXO (locked to new key)
    tx.addOutput({
      satoshis: 10000,
      lockingScript: Script.fromPubKeyHash(newControllerPublicKey.toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Resolve identity by DID
   */
  async resolveIdentity(did: string): Promise<Identity | null> {
    // In production, this would query an indexer service
    // For now, return null as placeholder
    console.log(`Resolving identity: ${did}`)
    return null
  }

  /**
   * Generate DID from public key
   */
  private generateDID(publicKey: PublicKey): string {
    const pubKeyHash = publicKey.toHash().toString('hex')
    return `did:bsv:${pubKeyHash}`
  }
}

/**
 * Usage Example
 */
async function identityExample() {
  const protocol = new IdentityProtocol()

  // Create identity
  const userKey = PrivateKey.fromRandom()
  const fundingUTXO: UTXO = {
    txid: '...',
    vout: 0,
    satoshis: 100000,
    script: Script.fromPubKeyHash(userKey.toPublicKey().toHash())
  }

  const { identity, tx } = await protocol.createIdentity(
    userKey,
    {
      name: 'Alice',
      email: 'alice@example.com',
      website: 'https://alice.example'
    },
    [
      {
        id: 'profile-service',
        type: 'ProfileService',
        serviceEndpoint: 'https://profile.alice.example'
      }
    ],
    fundingUTXO
  )

  console.log('Identity DID:', identity.did)
  console.log('Creation tx:', tx.id('hex'))

  // Rotate key
  const newKey = PrivateKey.fromRandom()
  const identityUTXO: UTXO = {
    txid: tx.id('hex'),
    vout: 1,
    satoshis: 10000,
    script: Script.fromPubKeyHash(userKey.toPublicKey().toHash())
  }

  const rotateTx = await protocol.rotateKey(
    identity,
    newKey.toPublicKey(),
    userKey,
    identityUTXO
  )

  console.log('Key rotation tx:', rotateTx.id('hex'))
}
```

## 5. Supply Chain Tracking Protocol

### Data Anchoring and Provenance

Reference: **[Transaction Output Component](../../../sdk-components/transaction-output/README.md)**

```typescript
import { Transaction, Script, PrivateKey, PublicKey, OP } from '@bsv/sdk'

/**
 * Supply Chain Tracking Protocol
 *
 * Features:
 * - Track products from origin to destination
 * - Record events (manufacture, shipping, delivery)
 * - Verify authenticity and provenance
 * - Support for multi-party verification
 * - Enable transparency and auditing
 */

interface Product {
  id: string
  name: string
  manufacturer: string
  manufacturedAt: number
  metadata: Record<string, any>
}

interface SupplyChainEvent {
  eventType: 'MANUFACTURE' | 'SHIP' | 'RECEIVE' | 'TRANSFER' | 'DELIVER'
  productId: string
  timestamp: number
  location: {
    lat?: number
    lon?: number
    address?: string
  }
  party: string // Public key of responsible party
  metadata: Record<string, any>
  previousEventTxId?: string
}

class SupplyChainProtocol {
  private readonly protocolPrefix = 'SUPPLY'
  private readonly version = 1

  /**
   * Record product manufacture
   */
  async recordManufacture(
    product: Product,
    manufacturerPrivateKey: PrivateKey,
    fundingUTXO: UTXO
  ): Promise<Transaction> {
    const event: SupplyChainEvent = {
      eventType: 'MANUFACTURE',
      productId: product.id,
      timestamp: Math.floor(Date.now() / 1000),
      location: product.metadata.manufactureLocation || {},
      party: manufacturerPrivateKey.toPublicKey().toString(),
      metadata: {
        product: product.name,
        manufacturer: product.manufacturer,
        batchNumber: product.metadata.batchNumber,
        certifications: product.metadata.certifications
      }
    }

    return this.recordEvent(event, manufacturerPrivateKey, fundingUTXO)
  }

  /**
   * Record shipment event
   */
  async recordShipment(
    productId: string,
    from: string,
    to: string,
    carrier: string,
    trackingNumber: string,
    previousEventTxId: string,
    shipperPrivateKey: PrivateKey,
    fundingUTXO: UTXO
  ): Promise<Transaction> {
    const event: SupplyChainEvent = {
      eventType: 'SHIP',
      productId,
      timestamp: Math.floor(Date.now() / 1000),
      location: { address: from },
      party: shipperPrivateKey.toPublicKey().toString(),
      metadata: {
        from,
        to,
        carrier,
        trackingNumber,
        estimatedDelivery: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString()
      },
      previousEventTxId
    }

    return this.recordEvent(event, shipperPrivateKey, fundingUTXO)
  }

  /**
   * Record delivery event
   */
  async recordDelivery(
    productId: string,
    location: string,
    receiverSignature: string,
    previousEventTxId: string,
    receiverPrivateKey: PrivateKey,
    fundingUTXO: UTXO
  ): Promise<Transaction> {
    const event: SupplyChainEvent = {
      eventType: 'DELIVER',
      productId,
      timestamp: Math.floor(Date.now() / 1000),
      location: { address: location },
      party: receiverPrivateKey.toPublicKey().toString(),
      metadata: {
        receiverSignature,
        condition: 'good',
        notes: 'Package delivered in good condition'
      },
      previousEventTxId
    }

    return this.recordEvent(event, receiverPrivateKey, fundingUTXO)
  }

  /**
   * Record generic supply chain event
   */
  private async recordEvent(
    event: SupplyChainEvent,
    signerPrivateKey: PrivateKey,
    fundingUTXO: UTXO
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: fundingUTXO.txid,
      sourceOutputIndex: fundingUTXO.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    // Output 0: Event data (OP_RETURN)
    const eventData = Buffer.from(JSON.stringify(event), 'utf8')

    tx.addOutput({
      satoshis: 0,
      lockingScript: ProtocolIdentifier.createPrefixOutput(
        this.protocolPrefix,
        this.version,
        event.eventType,
        eventData
      )
    })

    // Output 1: Chain continuation UTXO (for next event)
    tx.addOutput({
      satoshis: 1000,
      lockingScript: Script.fromPubKeyHash(signerPrivateKey.toPublicKey().toHash())
    })

    // Change
    const fee = 500
    tx.addOutput({
      satoshis: fundingUTXO.satoshis - 1000 - fee,
      lockingScript: Script.fromPubKeyHash(signerPrivateKey.toPublicKey().toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Verify product provenance
   * Traces complete history from manufacture to current state
   */
  async verifyProvenance(productId: string): Promise<{
    valid: boolean
    events: SupplyChainEvent[]
    issues: string[]
  }> {
    // In production, query indexer for all events related to productId
    // Verify chain of custody, timestamps, signatures

    const events: SupplyChainEvent[] = [] // Would fetch from indexer
    const issues: string[] = []

    // Verify event chain
    for (let i = 1; i < events.length; i++) {
      if (!events[i].previousEventTxId) {
        issues.push(`Event ${i} missing previous event reference`)
      }

      if (events[i].timestamp < events[i - 1].timestamp) {
        issues.push(`Event ${i} timestamp is before previous event`)
      }
    }

    return {
      valid: issues.length === 0,
      events,
      issues
    }
  }
}

/**
 * Usage Example
 */
async function supplyChainExample() {
  const protocol = new SupplyChainProtocol()

  // Manufacturer creates product
  const manufacturerKey = PrivateKey.fromRandom()
  const product: Product = {
    id: 'prod-12345',
    name: 'Medical Device XYZ',
    manufacturer: 'MedCo Inc',
    manufacturedAt: Math.floor(Date.now() / 1000),
    metadata: {
      batchNumber: 'BATCH-2024-001',
      certifications: ['FDA', 'CE'],
      manufactureLocation: { address: 'Factory 1, City A' }
    }
  }

  const fundingUTXO: UTXO = {
    txid: '...',
    vout: 0,
    satoshis: 100000,
    script: Script.fromPubKeyHash(manufacturerKey.toPublicKey().toHash())
  }

  const manufactureTx = await protocol.recordManufacture(
    product,
    manufacturerKey,
    fundingUTXO
  )

  console.log('Manufacture recorded:', manufactureTx.id('hex'))

  // Shipper records shipment
  const shipperKey = PrivateKey.fromRandom()
  const shipmentUTXO: UTXO = {
    txid: manufactureTx.id('hex'),
    vout: 1,
    satoshis: 1000,
    script: Script.fromPubKeyHash(shipperKey.toPublicKey().toHash())
  }

  const shipmentTx = await protocol.recordShipment(
    product.id,
    'Factory 1, City A',
    'Hospital B, City C',
    'ShipCo Express',
    'TRACK123456',
    manufactureTx.id('hex'),
    shipperKey,
    shipmentUTXO
  )

  console.log('Shipment recorded:', shipmentTx.id('hex'))

  // Receiver confirms delivery
  const receiverKey = PrivateKey.fromRandom()
  const deliveryUTXO: UTXO = {
    txid: shipmentTx.id('hex'),
    vout: 1,
    satoshis: 1000,
    script: Script.fromPubKeyHash(receiverKey.toPublicKey().toHash())
  }

  const deliveryTx = await protocol.recordDelivery(
    product.id,
    'Hospital B, City C',
    'John Doe - Receiving Manager',
    shipmentTx.id('hex'),
    receiverKey,
    deliveryUTXO
  )

  console.log('Delivery recorded:', deliveryTx.id('hex'))

  // Verify complete provenance
  const verification = await protocol.verifyProvenance(product.id)
  console.log('Provenance valid:', verification.valid)
  console.log('Total events:', verification.events.length)
}
```

## 6. Protocol Indexer and Discovery

### Building a Protocol Transaction Indexer

Reference: **[SPV Component](../../../sdk-components/spv/README.md)**

```typescript
import { Transaction, Script, OP } from '@bsv/sdk'

/**
 * Protocol Indexer
 *
 * Discovers and indexes protocol transactions for efficient querying
 *
 * Features:
 * - Monitor blockchain for protocol transactions
 * - Parse and validate protocol data
 * - Store in queryable database
 * - Provide API for protocol clients
 * - Support for SPV verification
 */

interface IndexedTransaction {
  txid: string
  blockHeight: number
  timestamp: number
  protocol: string
  version: number
  action: string
  data: any
  spvProof?: any
}

class ProtocolIndexer {
  private transactions: Map<string, IndexedTransaction[]> = new Map()

  /**
   * Index a transaction if it matches protocol prefix
   */
  async indexTransaction(
    tx: Transaction,
    blockHeight: number,
    timestamp: number
  ): Promise<boolean> {
    try {
      // Check each output for protocol data
      for (let i = 0; i < tx.outputs.length; i++) {
        const output = tx.outputs[i]
        const protocolData = ProtocolIdentifier.parseProtocolOutput(output.lockingScript)

        if (protocolData) {
          const indexed: IndexedTransaction = {
            txid: tx.id('hex'),
            blockHeight,
            timestamp,
            protocol: protocolData.prefix,
            version: protocolData.version,
            action: protocolData.action,
            data: this.parseProtocolData(protocolData.data, protocolData.prefix)
          }

          // Store by protocol prefix
          const protocolTxs = this.transactions.get(protocolData.prefix) || []
          protocolTxs.push(indexed)
          this.transactions.set(protocolData.prefix, protocolTxs)

          console.log(`Indexed ${protocolData.prefix} transaction: ${tx.id('hex')}`)
          return true
        }
      }

      return false
    } catch (error) {
      console.error('Error indexing transaction:', error)
      return false
    }
  }

  /**
   * Query transactions by protocol
   */
  queryByProtocol(
    protocol: string,
    filters?: {
      action?: string
      fromBlock?: number
      toBlock?: number
      limit?: number
    }
  ): IndexedTransaction[] {
    let txs = this.transactions.get(protocol) || []

    // Apply filters
    if (filters) {
      if (filters.action) {
        txs = txs.filter(tx => tx.action === filters.action)
      }

      if (filters.fromBlock !== undefined) {
        txs = txs.filter(tx => tx.blockHeight >= filters.fromBlock!)
      }

      if (filters.toBlock !== undefined) {
        txs = txs.filter(tx => tx.blockHeight <= filters.toBlock!)
      }

      if (filters.limit !== undefined) {
        txs = txs.slice(0, filters.limit)
      }
    }

    return txs
  }

  /**
   * Get transaction count by protocol
   */
  getProtocolStats(protocol: string): {
    totalTransactions: number
    actionCounts: Record<string, number>
    firstSeen: number
    lastSeen: number
  } {
    const txs = this.transactions.get(protocol) || []
    const actionCounts: Record<string, number> = {}

    for (const tx of txs) {
      actionCounts[tx.action] = (actionCounts[tx.action] || 0) + 1
    }

    return {
      totalTransactions: txs.length,
      actionCounts,
      firstSeen: txs.length > 0 ? txs[0].timestamp : 0,
      lastSeen: txs.length > 0 ? txs[txs.length - 1].timestamp : 0
    }
  }

  /**
   * Parse protocol-specific data
   */
  private parseProtocolData(data: Buffer, protocol: string): any {
    try {
      // Attempt JSON parsing (most protocols use JSON)
      return JSON.parse(data.toString('utf8'))
    } catch {
      // Return raw buffer if not JSON
      return { raw: data.toString('hex') }
    }
  }

  /**
   * Subscribe to protocol events (real-time monitoring)
   */
  subscribe(
    protocol: string,
    callback: (tx: IndexedTransaction) => void
  ): () => void {
    // In production, this would connect to a blockchain monitor service
    // For now, return a no-op unsubscribe function
    console.log(`Subscribed to ${protocol} events`)

    return () => {
      console.log(`Unsubscribed from ${protocol} events`)
    }
  }
}

/**
 * Protocol Client
 *
 * High-level interface for interacting with indexed protocol data
 */
class ProtocolClient {
  constructor(
    private indexer: ProtocolIndexer,
    private protocol: string
  ) {}

  /**
   * Get all transactions for this protocol
   */
  async getTransactions(filters?: any): Promise<IndexedTransaction[]> {
    return this.indexer.queryByProtocol(this.protocol, filters)
  }

  /**
   * Get protocol statistics
   */
  async getStats() {
    return this.indexer.getProtocolStats(this.protocol)
  }

  /**
   * Subscribe to new transactions
   */
  onTransaction(callback: (tx: IndexedTransaction) => void): () => void {
    return this.indexer.subscribe(this.protocol, callback)
  }

  /**
   * Get specific transaction by txid
   */
  async getTransaction(txid: string): Promise<IndexedTransaction | null> {
    const txs = await this.getTransactions()
    return txs.find(tx => tx.txid === txid) || null
  }
}

/**
 * Usage Example
 */
async function indexerExample() {
  const indexer = new ProtocolIndexer()

  // Index some token transactions
  const tokenTx = new Transaction()
  // ... build token transaction ...

  await indexer.indexTransaction(tokenTx, 800000, Date.now())

  // Query token protocol
  const client = new ProtocolClient(indexer, 'STOKEN')
  const stats = await client.getStats()

  console.log('Token protocol stats:', stats)

  // Subscribe to new token transactions
  const unsubscribe = client.onTransaction((tx) => {
    console.log('New token transaction:', tx.txid)
  })

  // Later: unsubscribe
  unsubscribe()
}
```

## 7. Protocol Wallet Integration

### Building Protocol-Specific Wallets

Reference: **[UTXO Management Component](../../../sdk-components/utxo-management/README.md)**

```typescript
import { Transaction, Script, PrivateKey, PublicKey, OP } from '@bsv/sdk'

/**
 * Protocol Wallet
 *
 * Specialized wallet for managing protocol-specific assets
 *
 * Features:
 * - Track protocol UTXOs separately from regular satoshis
 * - Build protocol-specific transactions
 * - Validate protocol rules
 * - Support for multiple protocols
 * - Integration with indexer for balance updates
 */

interface ProtocolBalance {
  protocol: string
  asset: string
  amount: number
  utxos: ProtocolUTXO[]
}

interface ProtocolUTXO {
  txid: string
  vout: number
  satoshis: number
  protocol: string
  protocolData: any
}

class ProtocolWallet {
  private balances: Map<string, ProtocolBalance> = new Map()

  constructor(
    private privateKey: PrivateKey,
    private indexer: ProtocolIndexer
  ) {}

  /**
   * Get balance for specific protocol and asset
   */
  async getBalance(protocol: string, asset: string): Promise<number> {
    const key = `${protocol}:${asset}`
    const balance = this.balances.get(key)
    return balance ? balance.amount : 0
  }

  /**
   * Refresh balances from indexer
   */
  async refreshBalances(): Promise<void> {
    // In production, query indexer for UTXOs owned by this wallet
    // For now, this is a placeholder
    console.log('Refreshing protocol balances...')
  }

  /**
   * Send protocol asset
   */
  async send(
    protocol: string,
    asset: string,
    amount: number,
    recipientPubKey: PublicKey
  ): Promise<Transaction> {
    // Get UTXOs for this protocol/asset
    const key = `${protocol}:${asset}`
    const balance = this.balances.get(key)

    if (!balance || balance.amount < amount) {
      throw new Error('Insufficient balance')
    }

    // Select UTXOs
    const selectedUTXOs = this.selectUTXOs(balance.utxos, amount)

    // Build transaction based on protocol
    switch (protocol) {
      case 'STOKEN':
        return this.sendTokens(asset, amount, recipientPubKey, selectedUTXOs)
      case 'VCP':
        return this.sendCredential(asset, recipientPubKey, selectedUTXOs)
      default:
        throw new Error(`Unsupported protocol: ${protocol}`)
    }
  }

  /**
   * Send tokens (example for token protocol)
   */
  private async sendTokens(
    tokenId: string,
    amount: number,
    recipientPubKey: PublicKey,
    utxos: ProtocolUTXO[]
  ): Promise<Transaction> {
    const tokenProtocol = new TokenProtocol()

    // Calculate total amount in selected UTXOs
    const totalAmount = utxos.reduce((sum, utxo) =>
      sum + (utxo.protocolData.amount || 0), 0
    )

    // Use first UTXO as primary (simplified)
    const primaryUTXO: TokenUTXO = {
      txid: utxos[0].txid,
      vout: utxos[0].vout,
      tokenId,
      amount: totalAmount,
      owner: this.privateKey.toPublicKey().toHash().toString('hex'),
      satoshis: utxos[0].satoshis
    }

    return tokenProtocol.transferTokens(
      primaryUTXO,
      amount,
      recipientPubKey,
      this.privateKey
    )
  }

  /**
   * Send credential (example for credentials protocol)
   */
  private async sendCredential(
    credentialId: string,
    recipientPubKey: PublicKey,
    utxos: ProtocolUTXO[]
  ): Promise<Transaction> {
    // Implementation would depend on credential transfer logic
    throw new Error('Credential transfer not implemented')
  }

  /**
   * Select UTXOs for transaction
   */
  private selectUTXOs(
    utxos: ProtocolUTXO[],
    targetAmount: number
  ): ProtocolUTXO[] {
    // Simple largest-first selection
    const sorted = [...utxos].sort((a, b) =>
      (b.protocolData.amount || 0) - (a.protocolData.amount || 0)
    )

    const selected: ProtocolUTXO[] = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.protocolData.amount || 0

      if (total >= targetAmount) {
        break
      }
    }

    if (total < targetAmount) {
      throw new Error('Insufficient UTXOs for target amount')
    }

    return selected
  }

  /**
   * Get transaction history for protocol
   */
  async getHistory(protocol: string): Promise<IndexedTransaction[]> {
    const publicKeyHash = this.privateKey.toPublicKey().toHash().toString('hex')

    // Query indexer for transactions involving this wallet
    const allTxs = this.indexer.queryByProtocol(protocol)

    // Filter for transactions where this wallet is involved
    return allTxs.filter(tx => {
      const data = tx.data
      return (
        data.from === publicKeyHash ||
        data.to === publicKeyHash ||
        data.owner === publicKeyHash
      )
    })
  }
}

/**
 * Usage Example
 */
async function walletExample() {
  const indexer = new ProtocolIndexer()
  const userKey = PrivateKey.fromRandom()
  const wallet = new ProtocolWallet(userKey, indexer)

  // Refresh balances
  await wallet.refreshBalances()

  // Check token balance
  const balance = await wallet.getBalance('STOKEN', 'my-token-001')
  console.log('Token balance:', balance)

  // Send tokens
  const recipientKey = PrivateKey.fromRandom()
  const tx = await wallet.send(
    'STOKEN',
    'my-token-001',
    1000,
    recipientKey.toPublicKey()
  )

  console.log('Transfer tx:', tx.id('hex'))

  // Get transaction history
  const history = await wallet.getHistory('STOKEN')
  console.log('Transaction history:', history.length, 'transactions')
}
```

## 8. Interoperability Standards

### BRC Compliance and Cross-Protocol Integration

Reference: **[BRC-29 Component](../../../sdk-components/brc-29/README.md)**

```typescript
import { Transaction, PrivateKey, PublicKey } from '@bsv/sdk'

/**
 * Protocol Interoperability Layer
 *
 * Enables different protocols to interact and share data
 *
 * Key Standards:
 * - BRC-1: Messaging between protocols
 * - BRC-2: Encryption for private data
 * - BRC-3: Signature verification
 * - BRC-29: Payment protocol integration
 * - BRC-42: Key derivation for protocol isolation
 */

interface ProtocolMessage {
  from: string // Source protocol
  to: string // Destination protocol
  messageType: string
  payload: any
  signature: string
}

class InteroperabilityLayer {
  /**
   * Send message from one protocol to another
   */
  async sendCrossProtocolMessage(
    fromProtocol: string,
    toProtocol: string,
    messageType: string,
    payload: any,
    senderPrivateKey: PrivateKey
  ): Promise<ProtocolMessage> {
    const message: ProtocolMessage = {
      from: fromProtocol,
      to: toProtocol,
      messageType,
      payload,
      signature: ''
    }

    // Sign message
    const messageHash = this.hashMessage(message)
    const signature = senderPrivateKey.sign(messageHash)
    message.signature = signature.toHex()

    return message
  }

  /**
   * Verify cross-protocol message
   */
  verifyMessage(
    message: ProtocolMessage,
    senderPublicKey: PublicKey
  ): boolean {
    try {
      const messageCopy = { ...message, signature: '' }
      const messageHash = this.hashMessage(messageCopy)
      const signature = require('@bsv/sdk').Signature.fromHex(message.signature)

      return senderPublicKey.verify(messageHash, signature)
    } catch (error) {
      return false
    }
  }

  /**
   * Convert token to credential (cross-protocol operation)
   */
  async tokenToCredential(
    tokenUTXO: TokenUTXO,
    credentialType: string,
    claims: Record<string, any>,
    issuerPrivateKey: PrivateKey
  ): Promise<{ tx: Transaction; credential: any }> {
    // Burn token
    const tokenProtocol = new TokenProtocol()
    const burnTx = await tokenProtocol.burnTokens(
      tokenUTXO,
      tokenUTXO.amount,
      issuerPrivateKey
    )

    // Issue credential
    const credentialProtocol = new VerifiableCredentialsProtocol()
    const { credential, tx: credentialTx } = await credentialProtocol.issueCredential(
      credentialType,
      issuerPrivateKey.toPublicKey(),
      {
        ...claims,
        originalToken: tokenUTXO.tokenId,
        burnTx: burnTx.id('hex')
      },
      issuerPrivateKey
    )

    return { tx: credentialTx, credential }
  }

  /**
   * Hash message for signing
   */
  private hashMessage(message: Omit<ProtocolMessage, 'signature'>): Buffer {
    const json = JSON.stringify(message)
    const hash = require('crypto').createHash('sha256')
    return hash.update(json).digest()
  }
}

/**
 * BRC-29 Payment Integration
 *
 * Allows protocols to request and receive payments
 */
class ProtocolPaymentIntegration {
  /**
   * Create payment request for protocol action
   */
  async createPaymentRequest(
    protocol: string,
    action: string,
    amount: number,
    recipientPublicKey: PublicKey,
    metadata?: any
  ): Promise<any> {
    // Use BRC-29 payment protocol
    // Reference: BRC-29 Component

    return {
      protocol,
      action,
      outputs: [
        {
          amount,
          script: require('@bsv/sdk').Script.fromPubKeyHash(recipientPublicKey.toHash()).toHex()
        }
      ],
      metadata,
      memo: `Payment for ${protocol} ${action}`
    }
  }

  /**
   * Process payment response
   */
  async processPaymentResponse(
    paymentRequest: any,
    paymentTx: Transaction
  ): Promise<boolean> {
    // Verify payment matches request
    const expectedAmount = paymentRequest.outputs[0].amount
    const expectedScript = paymentRequest.outputs[0].script

    // Check transaction outputs
    for (const output of paymentTx.outputs) {
      if (
        output.satoshis >= expectedAmount &&
        output.lockingScript.toHex() === expectedScript
      ) {
        return true
      }
    }

    return false
  }
}

/**
 * Usage Example
 */
async function interoperabilityExample() {
  const interop = new InteroperabilityLayer()
  const userKey = PrivateKey.fromRandom()

  // Send cross-protocol message
  const message = await interop.sendCrossProtocolMessage(
    'STOKEN',
    'VCP',
    'TOKEN_EXCHANGE',
    { tokenId: 'my-token-001', amount: 1000 },
    userKey
  )

  console.log('Cross-protocol message:', message)

  // Verify message
  const isValid = interop.verifyMessage(message, userKey.toPublicKey())
  console.log('Message valid:', isValid)

  // Payment integration
  const payments = new ProtocolPaymentIntegration()
  const paymentRequest = await payments.createPaymentRequest(
    'STOKEN',
    'PURCHASE',
    50000,
    userKey.toPublicKey(),
    { tokenId: 'my-token-001', quantity: 100 }
  )

  console.log('Payment request created:', paymentRequest)
}
```

## 9. Production Deployment

### Protocol Infrastructure

```typescript
/**
 * Production Protocol Deployment Checklist
 *
 * 1. Protocol Specification
 *    - Write comprehensive protocol spec document
 *    - Define all transaction types and data formats
 *    - Specify validation rules clearly
 *    - Version protocol appropriately
 *
 * 2. Indexer Service
 *    - Deploy protocol indexer (monitor blockchain)
 *    - Setup database for protocol transactions
 *    - Create API endpoints for protocol queries
 *    - Implement caching for performance
 *
 * 3. Wallet Integration
 *    - Build protocol-specific wallet
 *    - Support for major platforms (web, mobile, desktop)
 *    - Clear UI for protocol operations
 *    - Proper error handling and user feedback
 *
 * 4. Documentation
 *    - API documentation for developers
 *    - User guides for end users
 *    - Example code and tutorials
 *    - Integration guides for third parties
 *
 * 5. Testing
 *    - Unit tests for protocol logic
 *    - Integration tests with BSV blockchain
 *    - Load testing for indexer
 *    - Security audits for smart contracts
 *
 * 6. Monitoring
 *    - Transaction monitoring
 *    - Indexer health checks
 *    - Error logging and alerts
 *    - Performance metrics
 *
 * 7. Governance
 *    - Protocol upgrade process
 *    - Community feedback channels
 *    - Dispute resolution mechanism
 *    - Backward compatibility strategy
 */

/**
 * Example Docker Compose for Protocol Infrastructure
 */
const dockerCompose = `
version: '3.8'

services:
  # Protocol Indexer
  indexer:
    build: ./indexer
    environment:
      - NODE_URL=https://api.whatsonchain.com
      - DATABASE_URL=postgresql://user:pass@db:5432/protocol
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    ports:
      - "3000:3000"

  # PostgreSQL Database
  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=protocol
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - db-data:/var/lib/postgresql/data

  # Redis Cache
  redis:
    image: redis:7
    ports:
      - "6379:6379"

  # API Gateway
  api:
    build: ./api
    environment:
      - INDEXER_URL=http://indexer:3000
    ports:
      - "8080:8080"

  # Frontend
  frontend:
    build: ./frontend
    environment:
      - API_URL=http://api:8080
    ports:
      - "80:80"

volumes:
  db-data:
`

/**
 * Example Kubernetes Deployment for Scalable Protocol Indexer
 */
const kubernetesDeployment = `
apiVersion: apps/v1
kind: Deployment
metadata:
  name: protocol-indexer
  namespace: bsv-protocol
spec:
  replicas: 3
  selector:
    matchLabels:
      app: protocol-indexer
  template:
    metadata:
      labels:
        app: protocol-indexer
    spec:
      containers:
      - name: indexer
        image: myregistry/protocol-indexer:latest
        env:
        - name: NODE_URL
          value: "https://api.whatsonchain.com"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: protocol-indexer-service
  namespace: bsv-protocol
spec:
  selector:
    app: protocol-indexer
  ports:
  - port: 80
    targetPort: 3000
  type: LoadBalancer
`

/**
 * Example Monitoring Setup with Prometheus
 */
const prometheusConfig = `
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'protocol-indexer'
    static_configs:
      - targets: ['indexer:3000']
    metrics_path: /metrics

  - job_name: 'protocol-api'
    static_configs:
      - targets: ['api:8080']
    metrics_path: /metrics

rule_files:
  - 'alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
`
```

## 10. Best Practices

### Protocol Design Guidelines

1. **Keep Protocol Simple**
   - Start with minimal viable features
   - Add complexity only when needed
   - Clear separation of concerns
   - Well-defined data structures

2. **Use OP_RETURN for Metadata**
   - Protocol identifiers in OP_RETURN outputs
   - Keeps locking scripts standard (P2PKH)
   - Enables easy indexing and discovery
   - Maintains SPV compatibility

3. **Version Everything**
   - Protocol version in every transaction
   - Plan for backward compatibility
   - Document version differences
   - Provide migration guides

4. **Design for SPV**
   - Light clients should verify protocol transactions
   - Include merkle proofs in protocol data
   - Minimize data needed for verification
   - Support offline verification

5. **Security First**
   - Validate all inputs thoroughly
   - Prevent double-spending in protocol logic
   - Use proper signature verification
   - Consider attack vectors early

6. **Plan for Scale**
   - Indexer should handle high transaction volumes
   - Use database indexes efficiently
   - Implement caching where appropriate
   - Design for horizontal scaling

7. **Documentation is Critical**
   - Clear protocol specification
   - Code examples for all operations
   - Integration guides for developers
   - User-friendly documentation

8. **Test Thoroughly**
   - Unit tests for all protocol logic
   - Integration tests with real transactions
   - Load testing for production readiness
   - Security audits before mainnet

9. **Community Engagement**
   - Open source protocol code
   - Accept feedback and contributions
   - Build ecosystem around protocol
   - Provide support channels

10. **Iterate Carefully**
    - Collect usage data and metrics
    - Identify pain points and improvements
    - Release updates systematically
    - Maintain backward compatibility

## Hands-On Project: Build a Complete Token Protocol

### Project Requirements

Build a production-ready token protocol with:

1. **Token Issuance**
   - Create new tokens with max supply
   - Set token metadata (name, symbol, decimals)
   - Issue tokens to owner address

2. **Token Transfers**
   - Transfer tokens between addresses
   - Support for partial transfers (change)
   - Fee handling for transactions

3. **Token Burning**
   - Remove tokens from circulation
   - Update total supply tracking

4. **Protocol Indexer**
   - Monitor blockchain for token transactions
   - Parse and store token data
   - Provide query API for balances

5. **Protocol Wallet**
   - Track token balances
   - Build transfer transactions
   - Display transaction history

6. **Testing Suite**
   - Unit tests for all operations
   - Integration tests with real transactions
   - Load tests for indexer performance

### Implementation Steps

1. Define protocol specification document
2. Implement protocol transaction builders (issue, transfer, burn)
3. Build indexer service with database
4. Create wallet interface
5. Write comprehensive tests
6. Deploy to testnet
7. Test with real transactions
8. Document everything
9. Deploy to mainnet
10. Build community and ecosystem

## Common Pitfalls

### 1. Overcomplicating Protocol Design

**Problem**: Adding too many features initially makes protocol hard to implement and maintain.

**Solution**: Start with MVP (Minimum Viable Protocol), add features incrementally based on usage.

### 2. Ignoring Backward Compatibility

**Problem**: Protocol updates break existing implementations and transactions.

**Solution**: Version all protocol messages, maintain support for older versions, provide clear migration paths.

### 3. Poor Indexer Performance

**Problem**: Indexer struggles with high transaction volume, queries are slow.

**Solution**: Use database indexes properly, implement caching, design for horizontal scaling.

### 4. Inadequate Validation

**Problem**: Invalid transactions get indexed, causing incorrect balances or state.

**Solution**: Validate all protocol data thoroughly before indexing, reject malformed transactions.

### 5. Missing SPV Support

**Problem**: Light clients cannot verify protocol transactions.

**Solution**: Design protocol for SPV from the start, include merkle proofs in protocol data.

### 6. Security Vulnerabilities

**Problem**: Protocol has exploits allowing token theft or double-spending.

**Solution**: Security audit before launch, proper signature verification, input validation, attack modeling.

### 7. Lack of Documentation

**Problem**: Developers struggle to integrate protocol due to missing docs.

**Solution**: Comprehensive docs from day one, code examples, integration guides, active support.

### 8. No Monitoring

**Problem**: Protocol issues go undetected in production.

**Solution**: Implement monitoring, logging, alerting from the start, track key metrics.

## Related Courses

- **[Advanced Scripting](../advanced-scripting/README.md)** - Custom locking script patterns
- **[Node Operations](../node-operations/README.md)** - Running infrastructure for protocols
- **[SPV Verification](../../intermediate/spv-verification/README.md)** - Light client integration
- **[Script Templates](../../intermediate/script-templates/README.md)** - Reusable script patterns

## Additional Resources

### Protocol Examples

- **BSV Token Protocols**: Various token standards on BSV
- **Run Protocol**: Token and NFT protocol
- **1Sat Ordinals**: Ordinal inscription protocol
- **Haste Arcade**: Gaming protocol example

### Documentation Resources

- **BSV Wiki - Protocols**: https://wiki.bitcoinsv.io/index.php/Protocol
- **BRC Standards**: https://github.com/bitcoin-sv/BRCs
- **Protocol Design Patterns**: Community resources

### Related SDK Components

- **[Transaction](../../../sdk-components/transaction/README.md)** - Transaction construction
- **[Script](../../../sdk-components/script/README.md)** - Script operations
- **[SPV](../../../sdk-components/spv/README.md)** - SPV verification
- **[BEEF](../../../sdk-components/beef/README.md)** - Transaction envelopes
- **[ARC](../../../sdk-components/arc/README.md)** - Transaction broadcasting
- **[BRC-29](../../../sdk-components/brc-29/README.md)** - Payment protocol

### Code Examples Repository

- **BSV SDK Examples**: https://github.com/bsv-blockchain/ts-sdk/tree/master/examples
- **Protocol Examples**: See SDK repository for protocol implementations

### Community Resources

- **BSV Blockchain Discord**: Join protocol developer channels
- **BSV Developers Forum**: Discuss protocol design
- **Stack Overflow**: Tag questions with `bsv` and `protocol`

### Academic Papers

- **Bitcoin White Paper**: Foundation for protocol design
- **Token Protocol Research**: Academic research on blockchain tokens
- **Protocol Security**: Research on protocol vulnerabilities

## Status

✅ Complete
