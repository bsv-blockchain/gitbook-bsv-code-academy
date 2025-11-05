# Project 3: Supply Chain Digital Passports

**Blockchain-Based Product Verification System**

Build a complete supply chain tracking system with digital product passports that enable end-to-end verification of products from manufacture through retail to consumer. This project demonstrates multi-party workflows, immutable documentation, and tamper-evident verification using production-ready patterns.

---

## What You'll Build

A production-ready supply chain digital passport system featuring:

- Product lifecycle tracking (manufacture → distribution → retail → consumer)
- Multi-party verification workflows (manufacturer, distributor, retailer)
- Immutable documentation and certification storage
- QR code integration for physical product verification
- Tamper-evident digital seals
- Compliance documentation (certifications, inspections)
- Consumer verification interface
- Chain of custody audit trails

---

## Learning Objectives

By completing this project, you will learn:

- **Multi-signature workflows** - Multiple parties signing transactions
- **Chain of custody** - Tracking products through supply chain stages
- **Hash chains** - Creating tamper-evident verification records
- **QR code integration** - Linking physical products to blockchain data
- **Immutable audit trails** - Permanent verification history
- **OP_RETURN metadata** - Storing product information on-chain
- **Time-stamped verification** - Proving when events occurred
- **Overlay network concepts** - Organizing supply chain data

---

## Architecture Overview

### System Components

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                     │
│  - Product registration form                            │
│  - Verification dashboard                               │
│  - Consumer verification interface                      │
│  - QR scanner integration                               │
│  - WalletClient integration                             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Backend API (Node.js)                 │
│  - Product registration                                 │
│  - Verification service                                 │
│  - Chain of custody tracking                            │
│  - QR code generation                                   │
│  - Document management                                  │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Database (MongoDB)                    │
│  - Products                                             │
│  - Verifications                                        │
│  - Supply chain parties                                 │
│  - Documents                                            │
│  - Chain of custody records                             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                    BSV Blockchain                       │
│  - Product registration transactions                    │
│  - Verification transactions (multi-sig)                │
│  - Metadata (OP_RETURN)                                 │
│  - Time-stamped audit trail                             │
└─────────────────────────────────────────────────────────┘
```

### Digital Passport Protocol

**Product Registration:**
```
Transaction Structure:
├── Output 0: Product UTXO (P2PKH to manufacturer)
└── Output 1: Metadata (OP_RETURN)
    ├── Protocol: "SUPPLY-CHAIN-PASSPORT-V1"
    ├── Action: "REGISTER"
    ├── Product ID: "PROD-12345"
    ├── Product Name
    ├── Manufacturer Signature
    └── Initial Hash: SHA-256(product details)
```

**Verification (Multi-Party):**
```
Transaction Structure:
├── Input: Previous verification UTXO
├── Output 0: New verification UTXO (P2PKH to next party)
└── Output 1: Metadata (OP_RETURN)
    ├── Protocol: "SUPPLY-CHAIN-PASSPORT-V1"
    ├── Action: "VERIFY"
    ├── Product ID: "PROD-12345"
    ├── Verification Stage: "DISTRIBUTION"
    ├── Party Address & Signature
    ├── Previous Hash (links to prior verification)
    └── Timestamp
```

---

## Part 1: Backend Implementation

### Project Setup

```bash
mkdir supply-chain-passports
cd supply-chain-passports

npm init -y

# Install dependencies
npm install @bsv/sdk express mongodb dotenv qrcode
npm install -D typescript @types/node @types/express @types/qrcode ts-node

# Create structure
mkdir -p src/{models,services,routes,utils}
mkdir -p uploads/{documents,certifications}
```

### Models

```typescript
// src/models/Product.ts
import { ObjectId } from 'mongodb'

export enum ProductStatus {
  REGISTERED = 'REGISTERED',
  IN_PRODUCTION = 'IN_PRODUCTION',
  MANUFACTURED = 'MANUFACTURED',
  IN_DISTRIBUTION = 'IN_DISTRIBUTION',
  AT_RETAILER = 'AT_RETAILER',
  SOLD = 'SOLD',
  RECALLED = 'RECALLED'
}

export enum PartyRole {
  MANUFACTURER = 'MANUFACTURER',
  DISTRIBUTOR = 'DISTRIBUTOR',
  RETAILER = 'RETAILER',
  INSPECTOR = 'INSPECTOR'
}

export interface Product {
  _id?: ObjectId
  productId: string // Unique identifier

  // Product details
  name: string
  description: string
  sku: string
  batchNumber: string
  serialNumber?: string

  // Manufacturer
  manufacturerId: string
  manufacturerAddress: string
  manufacturerName: string
  manufactureDate: Date
  expiryDate?: Date

  // Specifications
  specifications: {
    weight?: string
    dimensions?: string
    materials?: string[]
    origin?: string
    [key: string]: any
  }

  // Digital seal (tamper detection)
  initialHash: string // SHA-256 of product details
  currentHash: string // Updated with each verification

  // Blockchain
  registrationTxid: string
  currentTxid: string // Latest verification TXID

  // Status
  status: ProductStatus
  currentLocation?: string
  currentParty?: string

  // QR Code
  qrCode: string // Base64 encoded QR code image
  qrData: string // Data embedded in QR code

  // Documents
  documents: Array<{
    type: string // 'certificate', 'inspection', 'test-report'
    name: string
    hash: string
    url: string
    uploadedBy: string
    uploadedAt: Date
  }>

  createdAt: Date
  updatedAt: Date
}

export interface Verification {
  _id?: ObjectId
  verificationId: string
  productId: string

  // Verification details
  stage: 'MANUFACTURE' | 'DISTRIBUTION' | 'RETAIL' | 'INSPECTION' | 'CUSTOM'
  stageName: string // Custom name for stage

  // Party performing verification
  partyId: string
  partyAddress: string
  partyName: string
  partyRole: PartyRole

  // Verification data
  verifiedAt: Date
  location?: string
  notes?: string

  // Chain of custody
  previousHash: string // Hash of previous verification
  currentHash: string // Hash of this verification
  previousTxid?: string // Previous verification transaction

  // Signatures (multi-party)
  signatures: Array<{
    partyAddress: string
    partyName: string
    signature: string
    signedAt: Date
  }>

  // Blockchain
  txid: string
  vout: number
  spent: boolean
  spentTxid?: string

  // Documents attached to this verification
  documents: Array<{
    type: string
    name: string
    hash: string
    url: string
  }>

  createdAt: Date
}

export interface Party {
  _id?: ObjectId
  partyId: string

  // Identity
  name: string
  role: PartyRole
  address: string // BSV address

  // Contact
  contactEmail: string
  contactPhone?: string
  location: string

  // Credentials
  certifications: Array<{
    type: string
    number: string
    issuer: string
    validUntil: Date
  }>

  // Status
  verified: boolean
  active: boolean

  createdAt: Date
  updatedAt: Date
}
```

### Product Registration Service

```typescript
// src/services/ProductRegistrationService.ts
import { PrivateKey, Transaction, P2PKH, Script, OP } from '@bsv/sdk'
import { Product, ProductStatus } from '../models/Product'
import { Database } from '../utils/database'
import * as crypto from 'crypto'
import * as QRCode from 'qrcode'

export class ProductRegistrationService {
  private db: Database

  constructor(db: Database) {
    this.db = db
  }

  /**
   * Register new product with digital passport
   */
  async registerProduct(params: {
    name: string
    description: string
    sku: string
    batchNumber: string
    serialNumber?: string
    manufacturerId: string
    manufacturerAddress: string
    manufacturerName: string
    manufactureDate: Date
    expiryDate?: Date
    specifications: any
    manufacturerPrivateKey: PrivateKey
  }): Promise<Product> {
    // 1. Generate unique product ID
    const productId = this.generateProductId(params.sku, params.batchNumber)

    // 2. Calculate initial hash (digital seal)
    const initialHash = this.calculateProductHash({
      productId,
      name: params.name,
      sku: params.sku,
      batchNumber: params.batchNumber,
      serialNumber: params.serialNumber,
      manufactureDate: params.manufactureDate
    })

    // 3. Generate QR code
    const qrData = this.createQRData(productId)
    const qrCode = await this.generateQRCode(qrData)

    // 4. Build registration transaction
    const tx = new Transaction()

    // Note: In production, add inputs from manufacturer's wallet here
    // await tx.addInput({
    //   sourceTXID: manufacturerUTXO.txid,
    //   sourceOutputIndex: manufacturerUTXO.vout,
    //   unlockingScriptTemplate: new P2PKH().unlock(params.manufacturerPrivateKey)
    // })

    // Add product UTXO output
    tx.addOutput({
      satoshis: 1000, // Dust amount
      lockingScript: new P2PKH().lock(params.manufacturerAddress)
    })

    // Add metadata output
    const metadata = {
      protocol: 'SUPPLY-CHAIN-PASSPORT-V1',
      action: 'REGISTER',
      productId,
      name: params.name,
      sku: params.sku,
      batchNumber: params.batchNumber,
      serialNumber: params.serialNumber,
      manufacturer: {
        id: params.manufacturerId,
        address: params.manufacturerAddress,
        name: params.manufacturerName
      },
      manufactureDate: params.manufactureDate.toISOString(),
      expiryDate: params.expiryDate?.toISOString(),
      specifications: params.specifications,
      initialHash,
      timestamp: Date.now()
    }

    const metadataScript = this.createMetadataScript(metadata)
    tx.addOutput({
      satoshis: 0,
      lockingScript: metadataScript
    })

    // 5. Calculate fee, sign and broadcast
    await tx.fee()
    await tx.sign()
    const broadcastResult = await tx.broadcast()
    const txid = broadcastResult.txid

    // 6. Create product record
    const product: Product = {
      productId,
      name: params.name,
      description: params.description,
      sku: params.sku,
      batchNumber: params.batchNumber,
      serialNumber: params.serialNumber,
      manufacturerId: params.manufacturerId,
      manufacturerAddress: params.manufacturerAddress,
      manufacturerName: params.manufacturerName,
      manufactureDate: params.manufactureDate,
      expiryDate: params.expiryDate,
      specifications: params.specifications,
      initialHash,
      currentHash: initialHash,
      registrationTxid: txid,
      currentTxid: txid,
      status: ProductStatus.MANUFACTURED,
      qrCode,
      qrData,
      documents: [],
      createdAt: new Date(),
      updatedAt: new Date()
    }

    const result = await this.db.insertProduct(product)
    product._id = result.insertedId

    console.log(`✅ Product registered: ${productId}`)
    console.log(`Registration TXID: ${txid}`)
    console.log(`QR Code generated for product verification`)

    return product
  }

  /**
   * Add document to product
   */
  async addDocument(params: {
    productId: string
    type: string
    name: string
    filePath: string
    uploadedBy: string
  }): Promise<void> {
    const product = await this.db.getProduct(params.productId)
    if (!product) throw new Error('Product not found')

    // Calculate file hash
    const hash = await this.hashFile(params.filePath)

    const document = {
      type: params.type,
      name: params.name,
      hash,
      url: params.filePath,
      uploadedBy: params.uploadedBy,
      uploadedAt: new Date()
    }

    await this.db.updateProduct(product._id!, {
      documents: [...product.documents, document],
      updatedAt: new Date()
    })

    console.log(`Document added to product ${params.productId}: ${params.name}`)
  }

  /**
   * Generate QR code for product
   */
  private async generateQRCode(data: string): Promise<string> {
    try {
      // Generate QR code as base64 data URL
      const qrCodeDataURL = await QRCode.toDataURL(data, {
        errorCorrectionLevel: 'H',
        type: 'image/png',
        width: 300,
        margin: 2
      })
      return qrCodeDataURL
    } catch (error) {
      console.error('QR code generation failed:', error)
      throw new Error('Failed to generate QR code')
    }
  }

  /**
   * Create QR code data (URL to verification page)
   */
  private createQRData(productId: string): string {
    const baseUrl = process.env.APP_URL || 'https://verify.example.com'
    return `${baseUrl}/verify/${productId}`
  }

  /**
   * Calculate product hash for tamper detection
   */
  private calculateProductHash(data: any): string {
    const dataString = JSON.stringify(data, Object.keys(data).sort())
    return crypto.createHash('sha256').update(dataString).digest('hex')
  }

  /**
   * Create OP_RETURN metadata script
   */
  private createMetadataScript(metadata: any): Script {
    const script = new Script()
    script.writeOpCode(OP.OP_FALSE) // OP_FALSE
    script.writeOpCode(OP.OP_RETURN) // OP_RETURN

    // Protocol prefix
    const protocolBuffer = Buffer.from('SUPPLY-CHAIN-PASSPORT-V1', 'utf8')
    script.writeBin(protocolBuffer)

    // Metadata JSON
    const metadataBuffer = Buffer.from(JSON.stringify(metadata), 'utf8')
    script.writeBin(metadataBuffer)

    return script
  }

  /**
   * Generate unique product ID
   */
  private generateProductId(sku: string, batchNumber: string): string {
    const timestamp = Date.now()
    const random = Math.random().toString(36).substring(2, 8).toUpperCase()
    return `PROD-${sku}-${batchNumber}-${random}`
  }

  /**
   * Hash file for verification
   */
  private async hashFile(filePath: string): Promise<string> {
    // In production: Actually hash the file contents
    // For demo: Generate hash from path
    return crypto.createHash('sha256').update(filePath).digest('hex')
  }
}
```

### Verification Service (Multi-Party)

```typescript
// src/services/VerificationService.ts
import { PrivateKey, Transaction, P2PKH, Script, Signature } from '@bsv/sdk'
import { Verification, Product, PartyRole } from '../models/Product'
import { Database } from '../utils/database'
import * as crypto from 'crypto'

export class VerificationService {
  private db: Database

  constructor(db: Database) {
    this.db = db
  }

  /**
   * Create verification record (with multi-party signatures)
   */
  async createVerification(params: {
    productId: string
    stage: 'MANUFACTURE' | 'DISTRIBUTION' | 'RETAIL' | 'INSPECTION' | 'CUSTOM'
    stageName: string
    partyId: string
    partyAddress: string
    partyName: string
    partyRole: PartyRole
    location?: string
    notes?: string
    partyPrivateKey: PrivateKey
    additionalSigners?: Array<{
      partyAddress: string
      partyName: string
      privateKey: PrivateKey
    }>
  }): Promise<Verification> {
    // 1. Get product
    const product = await this.db.getProduct(params.productId)
    if (!product) throw new Error('Product not found')

    // 2. Get previous verification (for hash chain)
    const previousVerifications = await this.db.getVerificationsByProduct(params.productId)
    const previousVerification = previousVerifications[previousVerifications.length - 1]

    const previousHash = previousVerification
      ? previousVerification.currentHash
      : product.initialHash

    const previousTxid = previousVerification?.txid

    // 3. Calculate current hash (including previous hash for chain)
    const verificationData = {
      productId: params.productId,
      stage: params.stage,
      stageName: params.stageName,
      partyAddress: params.partyAddress,
      verifiedAt: new Date(),
      location: params.location,
      previousHash
    }

    const currentHash = this.calculateVerificationHash(verificationData)

    // 4. Generate verification ID
    const verificationId = this.generateVerificationId(params.productId, params.stage)

    // 5. Collect signatures from all parties
    const signatures: Array<{
      partyAddress: string
      partyName: string
      signature: string
      signedAt: Date
    }> = []

    // Primary party signature
    const primarySig = this.signData(currentHash, params.partyPrivateKey)
    signatures.push({
      partyAddress: params.partyAddress,
      partyName: params.partyName,
      signature: primarySig,
      signedAt: new Date()
    })

    // Additional signers (for multi-party verification)
    if (params.additionalSigners) {
      for (const signer of params.additionalSigners) {
        const sig = this.signData(currentHash, signer.privateKey)
        signatures.push({
          partyAddress: signer.partyAddress,
          partyName: signer.partyName,
          signature: sig,
          signedAt: new Date()
        })
      }
    }

    // 6. Build verification transaction
    const tx = new Transaction()

    // Add input: spend previous verification UTXO (if exists)
    if (previousVerification && !previousVerification.spent) {
      tx.addInput({
        sourceTXID: previousVerification.txid,
        sourceOutputIndex: previousVerification.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.partyPrivateKey),
        sequence: 0xffffffff
      })
    }

    // Add verification UTXO output (passes to next party)
    tx.addOutput({
      satoshis: 1000,
      lockingScript: new P2PKH().lock(params.partyAddress)
    })

    // Add metadata output
    const metadata = {
      protocol: 'SUPPLY-CHAIN-PASSPORT-V1',
      action: 'VERIFY',
      productId: params.productId,
      verificationId,
      stage: params.stage,
      stageName: params.stageName,
      party: {
        id: params.partyId,
        address: params.partyAddress,
        name: params.partyName,
        role: params.partyRole
      },
      verifiedAt: new Date().toISOString(),
      location: params.location,
      notes: params.notes,
      previousHash,
      currentHash,
      previousTxid,
      signatures: signatures.map(s => ({
        address: s.partyAddress,
        signature: s.signature
      })),
      timestamp: Date.now()
    }

    const metadataScript = this.createMetadataScript(metadata)
    tx.addOutput({
      satoshis: 0,
      lockingScript: metadataScript
    })

    // 7. Sign transaction
    // Note: For multi-party signatures, each party needs to sign
    // For production: implement proper multi-signature coordination
    await tx.sign()

    // If additional signers, they sign too
    if (params.additionalSigners) {
      for (const signer of params.additionalSigners) {
        await tx.sign()
      }
    }

    // 8. Calculate fee and broadcast
    await tx.fee()
    const broadcastResult = await tx.broadcast()
    const txid = broadcastResult.txid

    // 9. Create verification record
    const verification: Verification = {
      verificationId,
      productId: params.productId,
      stage: params.stage,
      stageName: params.stageName,
      partyId: params.partyId,
      partyAddress: params.partyAddress,
      partyName: params.partyName,
      partyRole: params.partyRole,
      verifiedAt: new Date(),
      location: params.location,
      notes: params.notes,
      previousHash,
      currentHash,
      previousTxid,
      signatures,
      txid,
      vout: 0,
      spent: false,
      documents: [],
      createdAt: new Date()
    }

    const result = await this.db.insertVerification(verification)
    verification._id = result.insertedId

    // 10. Mark previous verification as spent
    if (previousVerification) {
      await this.db.updateVerification(previousVerification._id!, {
        spent: true,
        spentTxid: txid
      })
    }

    // 11. Update product
    await this.db.updateProduct(product._id!, {
      currentHash,
      currentTxid: txid,
      status: this.getProductStatusForStage(params.stage),
      currentLocation: params.location,
      currentParty: params.partyName,
      updatedAt: new Date()
    })

    console.log(`✅ Verification created: ${verificationId}`)
    console.log(`Stage: ${params.stageName}`)
    console.log(`TXID: ${txid}`)
    console.log(`Signatures: ${signatures.length} parties`)

    return verification
  }

  /**
   * Verify product integrity (check hash chain)
   */
  async verifyProductIntegrity(productId: string): Promise<{
    valid: boolean
    verifications: number
    issues: string[]
  }> {
    const product = await this.db.getProduct(productId)
    if (!product) throw new Error('Product not found')

    const verifications = await this.db.getVerificationsByProduct(productId)
    const issues: string[] = []

    // Check initial hash
    let expectedHash = product.initialHash

    // Verify each verification in chain
    for (let i = 0; i < verifications.length; i++) {
      const verification = verifications[i]

      // Check previous hash matches
      if (verification.previousHash !== expectedHash) {
        issues.push(`Verification ${i + 1} (${verification.verificationId}): Hash mismatch`)
      }

      // Update expected hash for next verification
      expectedHash = verification.currentHash
    }

    // Check product's current hash matches last verification
    if (product.currentHash !== expectedHash) {
      issues.push('Product current hash does not match verification chain')
    }

    return {
      valid: issues.length === 0,
      verifications: verifications.length,
      issues
    }
  }

  /**
   * Get full chain of custody for product
   */
  async getChainOfCustody(productId: string): Promise<Array<{
    verificationId: string
    stage: string
    stageName: string
    party: string
    partyRole: string
    location?: string
    verifiedAt: Date
    txid: string
    signatures: number
    hash: string
  }>> {
    const verifications = await this.db.getVerificationsByProduct(productId)

    return verifications.map(v => ({
      verificationId: v.verificationId,
      stage: v.stage,
      stageName: v.stageName,
      party: v.partyName,
      partyRole: v.partyRole,
      location: v.location,
      verifiedAt: v.verifiedAt,
      txid: v.txid,
      signatures: v.signatures.length,
      hash: v.currentHash
    }))
  }

  /**
   * Sign data with private key
   */
  private signData(data: string, privateKey: PrivateKey): string {
    const dataBuffer = Buffer.from(data, 'utf8')
    const signature = privateKey.sign(dataBuffer)
    return signature.toString('hex')
  }

  /**
   * Calculate verification hash
   */
  private calculateVerificationHash(data: any): string {
    const dataString = JSON.stringify(data, Object.keys(data).sort())
    return crypto.createHash('sha256').update(dataString).digest('hex')
  }

  /**
   * Create OP_RETURN metadata script
   */
  private createMetadataScript(metadata: any): Script {
    const script = new Script()
    script.writeOpCode(OP.OP_FALSE) // OP_FALSE
    script.writeOpCode(OP.OP_RETURN) // OP_RETURN

    const protocolBuffer = Buffer.from('SUPPLY-CHAIN-PASSPORT-V1', 'utf8')
    script.writeBin(protocolBuffer)

    const metadataBuffer = Buffer.from(JSON.stringify(metadata), 'utf8')
    script.writeBin(metadataBuffer)

    return script
  }

  /**
   * Generate verification ID
   */
  private generateVerificationId(productId: string, stage: string): string {
    const timestamp = Date.now()
    const random = Math.random().toString(36).substring(2, 8).toUpperCase()
    return `VER-${productId}-${stage}-${random}`
  }

  /**
   * Map verification stage to product status
   */
  private getProductStatusForStage(stage: string): any {
    const statusMap: Record<string, any> = {
      'MANUFACTURE': 'MANUFACTURED',
      'DISTRIBUTION': 'IN_DISTRIBUTION',
      'RETAIL': 'AT_RETAILER',
      'INSPECTION': 'MANUFACTURED'
    }
    return statusMap[stage] || 'MANUFACTURED'
  }
}
```

### Party Management Service

```typescript
// src/services/PartyService.ts
import { Party, PartyRole } from '../models/Product'
import { Database } from '../utils/database'

export class PartyService {
  private db: Database

  constructor(db: Database) {
    this.db = db
  }

  /**
   * Register supply chain party
   */
  async registerParty(params: {
    name: string
    role: PartyRole
    address: string
    contactEmail: string
    contactPhone?: string
    location: string
  }): Promise<Party> {
    const partyId = this.generatePartyId(params.role)

    const party: Party = {
      partyId,
      name: params.name,
      role: params.role,
      address: params.address,
      contactEmail: params.contactEmail,
      contactPhone: params.contactPhone,
      location: params.location,
      certifications: [],
      verified: false,
      active: true,
      createdAt: new Date(),
      updatedAt: new Date()
    }

    const result = await this.db.insertParty(party)
    party._id = result.insertedId

    console.log(`Party registered: ${partyId} (${params.name})`)
    return party
  }

  /**
   * Add certification to party
   */
  async addCertification(params: {
    partyId: string
    type: string
    number: string
    issuer: string
    validUntil: Date
  }): Promise<void> {
    const party = await this.db.getParty(params.partyId)
    if (!party) throw new Error('Party not found')

    const certification = {
      type: params.type,
      number: params.number,
      issuer: params.issuer,
      validUntil: params.validUntil
    }

    await this.db.updateParty(party._id!, {
      certifications: [...party.certifications, certification],
      updatedAt: new Date()
    })

    console.log(`Certification added to party ${params.partyId}`)
  }

  /**
   * Verify party
   */
  async verifyParty(partyId: string): Promise<void> {
    const party = await this.db.getParty(partyId)
    if (!party) throw new Error('Party not found')

    await this.db.updateParty(party._id!, {
      verified: true,
      updatedAt: new Date()
    })

    console.log(`Party verified: ${partyId}`)
  }

  /**
   * Get party by address
   */
  async getPartyByAddress(address: string): Promise<Party | null> {
    return await this.db.getPartyByAddress(address)
  }

  /**
   * Get all parties by role
   */
  async getPartiesByRole(role: PartyRole): Promise<Party[]> {
    return await this.db.getPartiesByRole(role)
  }

  private generatePartyId(role: PartyRole): string {
    const timestamp = Date.now()
    const random = Math.random().toString(36).substring(2, 8).toUpperCase()
    return `${role.substring(0, 3)}-${timestamp}-${random}`
  }
}
```

### API Routes

```typescript
// src/routes/products.ts
import { Router } from 'express'
import { ProductRegistrationService } from '../services/ProductRegistrationService'
import { VerificationService } from '../services/VerificationService'
import { PartyService } from '../services/PartyService'
import { Database } from '../utils/database'
import { PrivateKey } from '@bsv/sdk'

export function createProductRoutes(db: Database): Router {
  const router = Router()
  const productService = new ProductRegistrationService(db)
  const verificationService = new VerificationService(db)
  const partyService = new PartyService(db)

  // Register product
  router.post('/products/register', async (req, res) => {
    try {
      const { privateKeyWif, ...productData } = req.body
      const privateKey = PrivateKey.fromWif(privateKeyWif)

      const product = await productService.registerProduct({
        ...productData,
        manufacturerPrivateKey: privateKey
      })

      res.json({ success: true, product })
    } catch (error: any) {
      res.status(400).json({ success: false, error: error.message })
    }
  })

  // Get product
  router.get('/products/:productId', async (req, res) => {
    try {
      const product = await db.getProduct(req.params.productId)
      if (!product) {
        return res.status(404).json({ success: false, error: 'Product not found' })
      }
      res.json({ success: true, product })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Get all products
  router.get('/products', async (req, res) => {
    try {
      const products = await db.getAllProducts()
      res.json({ success: true, products })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Create verification
  router.post('/products/:productId/verify', async (req, res) => {
    try {
      const { privateKeyWif, additionalSigners, ...verificationData } = req.body
      const privateKey = PrivateKey.fromWif(privateKeyWif)

      // Parse additional signers if provided
      let parsedSigners
      if (additionalSigners) {
        parsedSigners = additionalSigners.map((s: any) => ({
          ...s,
          privateKey: PrivateKey.fromWif(s.privateKeyWif)
        }))
      }

      const verification = await verificationService.createVerification({
        productId: req.params.productId,
        ...verificationData,
        partyPrivateKey: privateKey,
        additionalSigners: parsedSigners
      })

      res.json({ success: true, verification })
    } catch (error: any) {
      res.status(400).json({ success: false, error: error.message })
    }
  })

  // Get product verifications (chain of custody)
  router.get('/products/:productId/verifications', async (req, res) => {
    try {
      const chainOfCustody = await verificationService.getChainOfCustody(req.params.productId)
      res.json({ success: true, chainOfCustody })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Verify product integrity
  router.get('/products/:productId/integrity', async (req, res) => {
    try {
      const integrity = await verificationService.verifyProductIntegrity(req.params.productId)
      res.json({ success: true, integrity })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Get product QR code
  router.get('/products/:productId/qr-code', async (req, res) => {
    try {
      const product = await db.getProduct(req.params.productId)
      if (!product) {
        return res.status(404).json({ success: false, error: 'Product not found' })
      }

      // Return QR code as image
      const base64Data = product.qrCode.replace(/^data:image\/png;base64,/, '')
      const img = Buffer.from(base64Data, 'base64')

      res.writeHead(200, {
        'Content-Type': 'image/png',
        'Content-Length': img.length
      })
      res.end(img)
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Register party
  router.post('/parties/register', async (req, res) => {
    try {
      const party = await partyService.registerParty(req.body)
      res.json({ success: true, party })
    } catch (error: any) {
      res.status(400).json({ success: false, error: error.message })
    }
  })

  // Get party
  router.get('/parties/:partyId', async (req, res) => {
    try {
      const party = await db.getParty(req.params.partyId)
      if (!party) {
        return res.status(404).json({ success: false, error: 'Party not found' })
      }
      res.json({ success: true, party })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  return router
}
```

### Main Server

```typescript
// src/index.ts
import express from 'express'
import * as dotenv from 'dotenv'
import { Database } from './utils/database'
import { createProductRoutes } from './routes/products'

dotenv.config()

async function main() {
  const app = express()
  app.use(express.json())

  // CORS for frontend
  app.use((req, res, next) => {
    res.header('Access-Control-Allow-Origin', '*')
    res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept')
    next()
  })

  // Connect to database
  const db = new Database(process.env.MONGODB_URI!)
  await db.connect()

  // Setup routes
  app.use('/api', createProductRoutes(db))

  // Health check
  app.get('/health', (req, res) => {
    res.json({ status: 'ok', service: 'supply-chain-passports' })
  })

  // Start server
  const PORT = process.env.PORT || 3000
  app.listen(PORT, () => {
    console.log(`🚀 Supply Chain Digital Passports API running on port ${PORT}`)
  })
}

main().catch(console.error)
```

---

## Part 2: Frontend Implementation

### Product Registration Form

```typescript
// src/components/ProductRegistrationForm.tsx
import React, { useState } from 'react'
import { useWallet } from '../hooks/useWallet'

export const ProductRegistrationForm: React.FC = () => {
  const { wallet, address } = useWallet()
  const [registering, setRegistering] = useState(false)

  const [formData, setFormData] = useState({
    name: '',
    description: '',
    sku: '',
    batchNumber: '',
    serialNumber: '',
    manufacturerName: '',
    manufactureDate: new Date().toISOString().split('T')[0],
    specifications: {
      weight: '',
      dimensions: '',
      materials: '',
      origin: ''
    }
  })

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    if (!wallet || !address) {
      alert('Please connect your wallet')
      return
    }

    setRegistering(true)

    try {
      // Get manufacturer ID (in production: from user session)
      const manufacturerId = 'MFG-001'

      // Request signature from wallet
      const messageToSign = `Register product: ${formData.name} (${formData.sku})`

      // In production: Get WIF from secure key management
      // For demo: User provides WIF
      const privateKeyWif = prompt('Enter your private key (WIF) to sign registration:')
      if (!privateKeyWif) throw new Error('Registration cancelled')

      const response = await fetch('/api/products/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...formData,
          manufacturerId,
          manufacturerAddress: address,
          manufactureDate: new Date(formData.manufactureDate),
          privateKeyWif
        })
      })

      const data = await response.json()

      if (!data.success) {
        throw new Error(data.error)
      }

      alert(`✅ Product registered successfully!\n\nProduct ID: ${data.product.productId}\nTXID: ${data.product.registrationTxid}`)

      // Show QR code
      window.open(`/api/products/${data.product.productId}/qr-code`, '_blank')

      // Reset form
      setFormData({
        name: '',
        description: '',
        sku: '',
        batchNumber: '',
        serialNumber: '',
        manufacturerName: '',
        manufactureDate: new Date().toISOString().split('T')[0],
        specifications: {
          weight: '',
          dimensions: '',
          materials: '',
          origin: ''
        }
      })

    } catch (error: any) {
      console.error('Registration failed:', error)
      alert(error.message)
    } finally {
      setRegistering(false)
    }
  }

  return (
    <div className="registration-form">
      <h2>Register Product</h2>

      <form onSubmit={handleSubmit}>
        <div className="form-section">
          <h3>Product Information</h3>

          <input
            type="text"
            placeholder="Product Name"
            value={formData.name}
            onChange={(e) => setFormData({ ...formData, name: e.target.value })}
            required
          />

          <textarea
            placeholder="Description"
            value={formData.description}
            onChange={(e) => setFormData({ ...formData, description: e.target.value })}
            required
          />

          <input
            type="text"
            placeholder="SKU"
            value={formData.sku}
            onChange={(e) => setFormData({ ...formData, sku: e.target.value })}
            required
          />

          <input
            type="text"
            placeholder="Batch Number"
            value={formData.batchNumber}
            onChange={(e) => setFormData({ ...formData, batchNumber: e.target.value })}
            required
          />

          <input
            type="text"
            placeholder="Serial Number (optional)"
            value={formData.serialNumber}
            onChange={(e) => setFormData({ ...formData, serialNumber: e.target.value })}
          />
        </div>

        <div className="form-section">
          <h3>Manufacturer Information</h3>

          <input
            type="text"
            placeholder="Manufacturer Name"
            value={formData.manufacturerName}
            onChange={(e) => setFormData({ ...formData, manufacturerName: e.target.value })}
            required
          />

          <input
            type="date"
            value={formData.manufactureDate}
            onChange={(e) => setFormData({ ...formData, manufactureDate: e.target.value })}
            required
          />
        </div>

        <div className="form-section">
          <h3>Specifications</h3>

          <input
            type="text"
            placeholder="Weight"
            value={formData.specifications.weight}
            onChange={(e) => setFormData({
              ...formData,
              specifications: { ...formData.specifications, weight: e.target.value }
            })}
          />

          <input
            type="text"
            placeholder="Dimensions"
            value={formData.specifications.dimensions}
            onChange={(e) => setFormData({
              ...formData,
              specifications: { ...formData.specifications, dimensions: e.target.value }
            })}
          />

          <input
            type="text"
            placeholder="Materials"
            value={formData.specifications.materials}
            onChange={(e) => setFormData({
              ...formData,
              specifications: { ...formData.specifications, materials: e.target.value }
            })}
          />

          <input
            type="text"
            placeholder="Country of Origin"
            value={formData.specifications.origin}
            onChange={(e) => setFormData({
              ...formData,
              specifications: { ...formData.specifications, origin: e.target.value }
            })}
          />
        </div>

        <button type="submit" disabled={registering}>
          {registering ? 'Registering...' : 'Register Product'}
        </button>
      </form>
    </div>
  )
}
```

### Verification Dashboard

```typescript
// src/components/VerificationDashboard.tsx
import React, { useState, useEffect } from 'react'
import { Product } from '../types'
import { useWallet } from '../hooks/useWallet'

export const VerificationDashboard: React.FC = () => {
  const { wallet, address } = useWallet()
  const [products, setProducts] = useState<Product[]>([])
  const [selectedProduct, setSelectedProduct] = useState<Product | null>(null)
  const [verifying, setVerifying] = useState(false)

  const [verificationData, setVerificationData] = useState({
    stage: 'DISTRIBUTION',
    stageName: '',
    partyName: '',
    partyRole: 'DISTRIBUTOR',
    location: '',
    notes: ''
  })

  useEffect(() => {
    loadProducts()
  }, [])

  const loadProducts = async () => {
    try {
      const response = await fetch('/api/products')
      const data = await response.json()
      setProducts(data.products)
    } catch (error) {
      console.error('Failed to load products:', error)
    }
  }

  const handleVerify = async () => {
    if (!wallet || !address || !selectedProduct) return

    setVerifying(true)

    try {
      // Get party ID (in production: from user session)
      const partyId = 'DIST-001'

      const privateKeyWif = prompt('Enter your private key (WIF) to sign verification:')
      if (!privateKeyWif) throw new Error('Verification cancelled')

      const response = await fetch(`/api/products/${selectedProduct.productId}/verify`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...verificationData,
          partyId,
          partyAddress: address,
          privateKeyWif
        })
      })

      const data = await response.json()

      if (!data.success) {
        throw new Error(data.error)
      }

      alert(`✅ Verification successful!\n\nVerification ID: ${data.verification.verificationId}\nTXID: ${data.verification.txid}`)

      // Reset form
      setVerificationData({
        stage: 'DISTRIBUTION',
        stageName: '',
        partyName: '',
        partyRole: 'DISTRIBUTOR',
        location: '',
        notes: ''
      })
      setSelectedProduct(null)
      loadProducts()

    } catch (error: any) {
      console.error('Verification failed:', error)
      alert(error.message)
    } finally {
      setVerifying(false)
    }
  }

  return (
    <div className="verification-dashboard">
      <h2>Verification Dashboard</h2>

      <div className="product-selector">
        <h3>Select Product to Verify</h3>
        <select
          onChange={(e) => {
            const product = products.find(p => p.productId === e.target.value)
            setSelectedProduct(product || null)
          }}
          value={selectedProduct?.productId || ''}
        >
          <option value="">-- Select Product --</option>
          {products.map(product => (
            <option key={product.productId} value={product.productId}>
              {product.name} ({product.productId})
            </option>
          ))}
        </select>
      </div>

      {selectedProduct && (
        <div className="verification-form">
          <h3>Create Verification</h3>

          <div className="product-info">
            <p><strong>Product:</strong> {selectedProduct.name}</p>
            <p><strong>SKU:</strong> {selectedProduct.sku}</p>
            <p><strong>Current Status:</strong> {selectedProduct.status}</p>
            <p><strong>Current Party:</strong> {selectedProduct.currentParty || 'None'}</p>
          </div>

          <select
            value={verificationData.stage}
            onChange={(e) => setVerificationData({ ...verificationData, stage: e.target.value as any })}
          >
            <option value="MANUFACTURE">Manufacture</option>
            <option value="DISTRIBUTION">Distribution</option>
            <option value="RETAIL">Retail</option>
            <option value="INSPECTION">Inspection</option>
            <option value="CUSTOM">Custom</option>
          </select>

          <input
            type="text"
            placeholder="Stage Name (e.g., 'Received at Warehouse')"
            value={verificationData.stageName}
            onChange={(e) => setVerificationData({ ...verificationData, stageName: e.target.value })}
          />

          <input
            type="text"
            placeholder="Party Name"
            value={verificationData.partyName}
            onChange={(e) => setVerificationData({ ...verificationData, partyName: e.target.value })}
          />

          <select
            value={verificationData.partyRole}
            onChange={(e) => setVerificationData({ ...verificationData, partyRole: e.target.value as any })}
          >
            <option value="MANUFACTURER">Manufacturer</option>
            <option value="DISTRIBUTOR">Distributor</option>
            <option value="RETAILER">Retailer</option>
            <option value="INSPECTOR">Inspector</option>
          </select>

          <input
            type="text"
            placeholder="Location"
            value={verificationData.location}
            onChange={(e) => setVerificationData({ ...verificationData, location: e.target.value })}
          />

          <textarea
            placeholder="Notes"
            value={verificationData.notes}
            onChange={(e) => setVerificationData({ ...verificationData, notes: e.target.value })}
          />

          <button onClick={handleVerify} disabled={verifying}>
            {verifying ? 'Creating Verification...' : 'Create Verification'}
          </button>
        </div>
      )}
    </div>
  )
}
```

### Consumer Verification Interface

```typescript
// src/components/ConsumerVerification.tsx
import React, { useState } from 'react'
import { Product } from '../types'

export const ConsumerVerification: React.FC = () => {
  const [productId, setProductId] = useState('')
  const [product, setProduct] = useState<Product | null>(null)
  const [chainOfCustody, setChainOfCustody] = useState<any[]>([])
  const [integrity, setIntegrity] = useState<any>(null)
  const [loading, setLoading] = useState(false)

  const handleVerify = async () => {
    if (!productId) return

    setLoading(true)

    try {
      // Load product
      const productResponse = await fetch(`/api/products/${productId}`)
      const productData = await productResponse.json()

      if (!productData.success) {
        throw new Error('Product not found')
      }

      setProduct(productData.product)

      // Load chain of custody
      const chainResponse = await fetch(`/api/products/${productId}/verifications`)
      const chainData = await chainResponse.json()
      setChainOfCustody(chainData.chainOfCustody)

      // Verify integrity
      const integrityResponse = await fetch(`/api/products/${productId}/integrity`)
      const integrityData = await integrityResponse.json()
      setIntegrity(integrityData.integrity)

    } catch (error: any) {
      console.error('Verification failed:', error)
      alert(error.message)
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="consumer-verification">
      <h2>Verify Product</h2>

      <div className="search-box">
        <input
          type="text"
          placeholder="Enter Product ID or scan QR code"
          value={productId}
          onChange={(e) => setProductId(e.target.value)}
        />
        <button onClick={handleVerify} disabled={loading}>
          {loading ? 'Verifying...' : 'Verify'}
        </button>
      </div>

      {product && (
        <div className="product-details">
          <h3>Product Information</h3>

          <div className="info-grid">
            <div>
              <strong>Name:</strong> {product.name}
            </div>
            <div>
              <strong>SKU:</strong> {product.sku}
            </div>
            <div>
              <strong>Batch:</strong> {product.batchNumber}
            </div>
            {product.serialNumber && (
              <div>
                <strong>Serial:</strong> {product.serialNumber}
              </div>
            )}
            <div>
              <strong>Manufacturer:</strong> {product.manufacturerName}
            </div>
            <div>
              <strong>Manufacture Date:</strong> {new Date(product.manufactureDate).toLocaleDateString()}
            </div>
            <div>
              <strong>Status:</strong> {product.status}
            </div>
            <div>
              <strong>Current Location:</strong> {product.currentLocation || 'Unknown'}
            </div>
          </div>

          {integrity && (
            <div className={`integrity-check ${integrity.valid ? 'valid' : 'invalid'}`}>
              <h4>
                {integrity.valid ? '✅ Product Verified - Authentic' : '❌ Verification Failed'}
              </h4>
              <p>Verifications: {integrity.verifications}</p>
              {integrity.issues.length > 0 && (
                <div className="issues">
                  <strong>Issues:</strong>
                  <ul>
                    {integrity.issues.map((issue: string, index: number) => (
                      <li key={index}>{issue}</li>
                    ))}
                  </ul>
                </div>
              )}
            </div>
          )}

          <div className="qr-code">
            <h4>Product QR Code</h4>
            <img src={product.qrCode} alt="Product QR Code" />
          </div>
        </div>
      )}

      {chainOfCustody.length > 0 && (
        <div className="chain-of-custody">
          <h3>Chain of Custody</h3>

          <div className="timeline">
            {chainOfCustody.map((verification, index) => (
              <div key={verification.verificationId} className="timeline-item">
                <div className="timeline-marker">{index + 1}</div>
                <div className="timeline-content">
                  <div className="stage-name">{verification.stageName}</div>
                  <div className="party-info">
                    <strong>{verification.party}</strong> ({verification.partyRole})
                  </div>
                  {verification.location && (
                    <div className="location">📍 {verification.location}</div>
                  )}
                  <div className="timestamp">
                    {new Date(verification.verifiedAt).toLocaleString()}
                  </div>
                  <div className="signatures">
                    {verification.signatures} signature(s)
                  </div>
                  <div className="txid">
                    <a
                      href={`https://whatsonchain.com/tx/${verification.txid}`}
                      target="_blank"
                      rel="noreferrer"
                    >
                      View on blockchain →
                    </a>
                  </div>
                </div>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  )
}
```

### QR Code Scanner Integration

```typescript
// src/components/QRScanner.tsx
import React, { useState, useRef, useEffect } from 'react'
import { useNavigate } from 'react-router-dom'

export const QRScanner: React.FC = () => {
  const [scanning, setScanning] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const videoRef = useRef<HTMLVideoElement>(null)
  const navigate = useNavigate()

  const startScanning = async () => {
    try {
      setError(null)
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: 'environment' }
      })

      if (videoRef.current) {
        videoRef.current.srcObject = stream
        setScanning(true)
      }
    } catch (err) {
      setError('Camera access denied or not available')
      console.error(err)
    }
  }

  const stopScanning = () => {
    if (videoRef.current && videoRef.current.srcObject) {
      const stream = videoRef.current.srcObject as MediaStream
      stream.getTracks().forEach(track => track.stop())
      setScanning(false)
    }
  }

  const handleScanResult = (productId: string) => {
    stopScanning()
    navigate(`/verify/${productId}`)
  }

  useEffect(() => {
    return () => {
      stopScanning()
    }
  }, [])

  return (
    <div className="qr-scanner">
      <h2>Scan Product QR Code</h2>

      {!scanning && (
        <button onClick={startScanning}>
          Start Camera
        </button>
      )}

      {scanning && (
        <div className="scanner-container">
          <video ref={videoRef} autoPlay playsInline />
          <button onClick={stopScanning}>
            Stop Scanning
          </button>
        </div>
      )}

      {error && (
        <div className="error">
          {error}
        </div>
      )}

      <div className="manual-entry">
        <p>Or enter Product ID manually:</p>
        <input
          type="text"
          placeholder="PROD-XXX-XXX-XXXX"
          onKeyPress={(e) => {
            if (e.key === 'Enter') {
              const productId = (e.target as HTMLInputElement).value
              handleScanResult(productId)
            }
          }}
        />
      </div>
    </div>
  )
}
```

---

## Testing

### Test Product Registration

```typescript
// tests/product-registration.test.ts
import { ProductRegistrationService } from '../src/services/ProductRegistrationService'
import { PrivateKey } from '@bsv/sdk'

describe('Product Registration', () => {
  it('should register product with QR code', async () => {
    const service = new ProductRegistrationService(db)
    const manufacturerKey = PrivateKey.fromRandom()

    const product = await service.registerProduct({
      name: 'Test Product',
      description: 'Testing supply chain',
      sku: 'TEST-001',
      batchNumber: 'BATCH-001',
      manufacturerId: 'MFG-001',
      manufacturerAddress: manufacturerKey.toPublicKey().toAddress(),
      manufacturerName: 'Test Manufacturer',
      manufactureDate: new Date(),
      specifications: {
        weight: '1kg',
        origin: 'USA'
      },
      manufacturerPrivateKey: manufacturerKey
    })

    expect(product.productId).toBeDefined()
    expect(product.registrationTxid).toBeDefined()
    expect(product.qrCode).toBeDefined()
    expect(product.initialHash).toBeDefined()
  })
})
```

### Test Verification Chain

```typescript
// tests/verification-chain.test.ts
describe('Verification Chain', () => {
  it('should create multi-party verification chain', async () => {
    // Register product
    const product = await productService.registerProduct({...})

    // Verification 1: Distributor
    const verification1 = await verificationService.createVerification({
      productId: product.productId,
      stage: 'DISTRIBUTION',
      stageName: 'Received at Warehouse',
      partyId: 'DIST-001',
      partyAddress: distributorAddress,
      partyName: 'ABC Distribution',
      partyRole: 'DISTRIBUTOR',
      partyPrivateKey: distributorKey
    })

    expect(verification1.txid).toBeDefined()
    expect(verification1.previousHash).toBe(product.initialHash)

    // Verification 2: Retailer
    const verification2 = await verificationService.createVerification({
      productId: product.productId,
      stage: 'RETAIL',
      stageName: 'Received at Store',
      partyId: 'RET-001',
      partyAddress: retailerAddress,
      partyName: 'XYZ Store',
      partyRole: 'RETAILER',
      partyPrivateKey: retailerKey
    })

    expect(verification2.previousHash).toBe(verification1.currentHash)

    // Verify integrity
    const integrity = await verificationService.verifyProductIntegrity(product.productId)
    expect(integrity.valid).toBe(true)
    expect(integrity.verifications).toBe(2)
  })
})
```

### Test Multi-Signature Verification

```typescript
// tests/multi-signature.test.ts
describe('Multi-Signature Verification', () => {
  it('should require multiple parties to sign', async () => {
    const verification = await verificationService.createVerification({
      productId: product.productId,
      stage: 'INSPECTION',
      stageName: 'Quality Inspection',
      partyId: 'INSP-001',
      partyAddress: inspectorAddress,
      partyName: 'Quality Inspector',
      partyRole: 'INSPECTOR',
      partyPrivateKey: inspectorKey,
      additionalSigners: [
        {
          partyAddress: managerAddress,
          partyName: 'Quality Manager',
          privateKey: managerKey
        }
      ]
    })

    expect(verification.signatures.length).toBe(2)
    expect(verification.signatures[0].partyAddress).toBe(inspectorAddress)
    expect(verification.signatures[1].partyAddress).toBe(managerAddress)
  })
})
```

### Test Hash Chain Integrity

```typescript
// tests/hash-chain.test.ts
describe('Hash Chain Integrity', () => {
  it('should detect tampering in verification chain', async () => {
    // Create valid chain
    const product = await productService.registerProduct({...})
    const verification1 = await verificationService.createVerification({...})
    const verification2 = await verificationService.createVerification({...})

    // Verify valid chain
    let integrity = await verificationService.verifyProductIntegrity(product.productId)
    expect(integrity.valid).toBe(true)

    // Tamper with verification (simulate database modification)
    await db.updateVerification(verification1._id!, {
      currentHash: 'tampered-hash'
    })

    // Verify chain now fails
    integrity = await verificationService.verifyProductIntegrity(product.productId)
    expect(integrity.valid).toBe(false)
    expect(integrity.issues.length).toBeGreaterThan(0)
  })
})
```

---

## Deployment Considerations

### Environment Variables

```env
# Network
NETWORK=mainnet

# Database
MONGODB_URI=mongodb://localhost:27017/supply-chain

# Application
APP_URL=https://verify.yourcompany.com
PORT=3000

# Security
JWT_SECRET=your-secret-key
ENCRYPTION_KEY=your-encryption-key
```

### Security Best Practices

**Private Key Management:**
- Use HSM (Hardware Security Module) for party keys
- Encrypt private keys at rest
- Implement key rotation policies
- Use multi-signature for high-value verifications

**Data Protection:**
- Encrypt sensitive documents
- Implement access control
- Audit all verification actions
- Rate limiting on API endpoints

**Tamper Detection:**
- Regular hash chain integrity checks
- Monitor for blockchain reorganizations
- Alert on verification anomalies
- Backup verification records

### Production Enhancements

**Overlay Network Integration:**
- Use overlay services for efficient data queries
- Index verifications by product, party, stage
- Real-time verification notifications
- Historical data archival

**Scalability:**
- Cache frequently accessed products
- Queue verification processing
- Optimize database indexes
- CDN for QR codes and images

**Monitoring:**
- Track verification latency
- Monitor hash chain integrity
- Alert on failed verifications
- Dashboard for supply chain metrics

---

## Summary

You've built a complete supply chain digital passports system demonstrating:

✅ **Product registration** - With QR codes and digital seals
✅ **Multi-party verification** - Multiple signatures per verification
✅ **Chain of custody** - Complete audit trail
✅ **Hash chains** - Tamper-evident verification records
✅ **Immutable documentation** - Blockchain-based certificates
✅ **Consumer verification** - Public verification interface
✅ **Production patterns** - Security, integrity, scalability

**Key Takeaways:**

1. **Multi-signature workflows** enable trust between parties
2. **Hash chains** provide tamper detection
3. **QR codes** link physical products to blockchain records
4. **OP_RETURN metadata** stores verification data on-chain
5. **Time-stamped verifications** prove when events occurred
6. **Immutable audit trails** enable complete provenance tracking

---

## Next Steps

**Enhance Your Implementation:**
- Add IoT sensor integration (temperature, location tracking)
- Implement multi-party approval workflows
- Add automated compliance checks
- Create mobile app for field verifications
- Integrate with ERP systems

**Scale Your System:**
- Deploy overlay network for efficient queries
- Implement advanced caching strategies
- Add real-time notification system
- Build analytics dashboard

**Deploy to Production:**
- Set up HSM for key management
- Configure monitoring and alerts
- Implement backup and recovery
- Deploy to cloud infrastructure

**Continue Learning:**
- [Advanced Multi-Signature](../../../code-features/multi-signature/)
- [Overlay Networks](../../../sdk-components/overlay-networks/)
- [Enterprise Deployment](../../../deployment/enterprise/)

---

## Resources

- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk/)
- [Transaction Building](../../../sdk-components/transaction/)
- [Script Templates](../../../sdk-components/script-templates/)
- [OP_RETURN Metadata](../../../code-features/op-return/)
- [Digital Signatures](../../../code-features/signatures/)
- [Supply Chain Use Cases](https://wiki.bitcoinsv.io/use-cases/supply-chain)

---

**Congratulations!** You've completed the Intermediate Learning Path. You now have the skills to build production-ready BSV applications with complex workflows, multi-party coordination, and real-world integration patterns.

Ready for more? Explore the [Advanced Learning Path](../../advanced/) for enterprise-scale systems, advanced cryptography, and protocol design.
