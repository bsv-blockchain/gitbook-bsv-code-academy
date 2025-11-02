# Project 1: Crowdfunding Platform

**Full-Stack BSV Application**

Build a complete crowdfunding platform where users can create campaigns, pledge funds, and receive automated payouts when goals are met. This project demonstrates real-world BSV application development with both backend and frontend implementations.

---

## What You'll Build

A production-ready crowdfunding platform featuring:

- ✅ Campaign creation with funding goals and deadlines
- ✅ User pledges with BSV payments
- ✅ Escrow mechanism holding funds until goal is met
- ✅ Automated payout to campaign creator
- ✅ Refunds if campaign fails
- ✅ Real-time campaign progress tracking
- ✅ Transaction history and receipts

---

## Learning Objectives

By completing this project, you will learn:

- **Escrow patterns** - Holding funds securely until conditions are met
- **Time-locked transactions** - Using nLockTime for refunds
- **Multi-output transactions** - Distributing funds to multiple recipients
- **State management** - Tracking campaign status on-chain and off-chain
- **Event-driven architecture** - Responding to blockchain events
- **Both paradigms** - Backend custodial and frontend non-custodial implementations

---

## Architecture Overview

### System Components

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                     │
│  - Campaign dashboard                                   │
│  - Create campaign form                                 │
│  - Pledge interface                                     │
│  - WalletClient integration (user wallets)             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Backend API (Node.js)                 │
│  - Campaign management                                  │
│  - Escrow service (holds funds)                        │
│  - Payout automation                                    │
│  - Transaction monitoring                               │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Database (MongoDB)                    │
│  - Campaigns                                            │
│  - Pledges                                              │
│  - Users                                                │
│  - Escrow UTXOs                                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                    BSV Blockchain                       │
│  - Pledge transactions                                  │
│  - Payout transactions                                  │
│  - Refund transactions                                  │
└─────────────────────────────────────────────────────────┘
```

### Data Flow

**Campaign Creation:**
```
Creator → Frontend → API → Database → Campaign ID
```

**Pledge Flow:**
```
Backer → WalletClient → Escrow Address → Database Updates → Campaign Progress
```

**Payout Flow (Goal Met):**
```
Cron Job → Check Goals → Build Payout TX → Broadcast → Creator Receives Funds
```

**Refund Flow (Goal Failed):**
```
Deadline Passes → Build Refund TXs → Broadcast → Backers Receive Refunds
```

---

## Part 1: Backend Implementation

### Project Setup

```bash
mkdir crowdfunding-platform
cd crowdfunding-platform

# Initialize project
npm init -y

# Install dependencies
npm install @bsv/sdk express mongodb dotenv
npm install -D typescript @types/node @types/express ts-node

# Create structure
mkdir -p src/{models,services,routes,utils}
```

### Campaign Model

```typescript
// src/models/Campaign.ts
import { ObjectId } from 'mongodb'

export enum CampaignStatus {
  ACTIVE = 'ACTIVE',
  FUNDED = 'FUNDED',
  FAILED = 'FAILED',
  PAID_OUT = 'PAID_OUT'
}

export interface Campaign {
  _id?: ObjectId
  creatorId: string
  creatorAddress: string
  title: string
  description: string
  goalAmount: number // satoshis
  currentAmount: number // satoshis
  deadline: Date
  status: CampaignStatus
  escrowAddress: string
  escrowPrivateKey: string // Encrypted in production!
  createdAt: Date
  updatedAt: Date
}

export interface Pledge {
  _id?: ObjectId
  campaignId: ObjectId
  backerId: string
  backerAddress: string
  amount: number // satoshis
  txid: string
  refunded: boolean
  refundTxid?: string
  createdAt: Date
}
```

### Escrow Service

```typescript
// src/services/EscrowService.ts
import { PrivateKey, Transaction, P2PKH } from '@bsv/sdk'
import { Campaign, Pledge } from '../models/Campaign'
import { Database } from '../utils/database'

export class EscrowService {
  private db: Database

  constructor(db: Database) {
    this.db = db
  }

  /**
   * Create unique escrow address for campaign
   */
  async createEscrowWallet(campaign: Campaign): Promise<{
    address: string
    privateKey: string // Should be encrypted before storing!
  }> {
    // Generate unique private key for this campaign's escrow
    const privateKey = PrivateKey.fromRandom()
    const address = privateKey.toPublicKey().toAddress()

    return {
      address,
      privateKey: privateKey.toWif() // Encrypt this before storing!
    }
  }

  /**
   * Monitor blockchain for incoming pledges to escrow address
   */
  async monitorEscrowAddress(campaignId: string, escrowAddress: string) {
    // In production: Use overlay service or blockchain monitor
    // For demo: Check periodically

    const pledges = await this.db.getPledgesForCampaign(campaignId)

    // Verify each pledge transaction exists and is confirmed
    for (const pledge of pledges) {
      if (!pledge.txid) continue

      // Check transaction status (use overlay service in production)
      const confirmed = await this.checkTransactionConfirmed(pledge.txid)

      if (confirmed) {
        console.log(`Pledge ${pledge.txid} confirmed for campaign ${campaignId}`)
      }
    }
  }

  /**
   * Process payout when campaign goal is met
   */
  async processPayout(campaign: Campaign): Promise<string> {
    // 1. Load escrow private key (decrypt first in production!)
    const escrowKey = PrivateKey.fromWif(campaign.escrowPrivateKey)

    // 2. Get all pledges for this campaign
    const pledges = await this.db.getPledgesForCampaign(campaign._id!.toString())
    const totalAmount = pledges.reduce((sum, p) => sum + p.amount, 0)

    // 3. Build payout transaction
    const tx = new Transaction()

    // 4. Add escrow output as input (spending the pledges)
    // Note: In practice, track escrow UTXOs in database
    // This is simplified for demonstration

    // 5. Add output to campaign creator
    const platformFee = Math.floor(totalAmount * 0.05) // 5% fee
    const creatorPayout = totalAmount - platformFee

    tx.addOutput({
      satoshis: creatorPayout,
      lockingScript: new P2PKH().lock(campaign.creatorAddress)
    })

    // 6. Add platform fee output
    const platformAddress = process.env.PLATFORM_ADDRESS!
    tx.addOutput({
      satoshis: platformFee,
      lockingScript: new P2PKH().lock(platformAddress)
    })

    // 7. Sign with escrow key
    await tx.sign(escrowKey)

    // 8. Broadcast
    const txid = await tx.broadcast()

    // 9. Update campaign status
    await this.db.updateCampaign(campaign._id!, {
      status: 'PAID_OUT',
      updatedAt: new Date()
    })

    console.log(`Campaign ${campaign._id} paid out. TXID: ${txid}`)
    return txid
  }

  /**
   * Process refunds when campaign fails
   */
  async processRefunds(campaign: Campaign): Promise<string[]> {
    const escrowKey = PrivateKey.fromWif(campaign.escrowPrivateKey)
    const pledges = await this.db.getPledgesForCampaign(campaign._id!.toString())

    const refundTxids: string[] = []

    // Create refund transaction for each backer
    for (const pledge of pledges) {
      if (pledge.refunded) continue

      const tx = new Transaction()

      // Add refund output to backer
      tx.addOutput({
        satoshis: pledge.amount,
        lockingScript: new P2PKH().lock(pledge.backerAddress)
      })

      // Sign and broadcast
      await tx.sign(escrowKey)
      const txid = await tx.broadcast()

      // Update pledge as refunded
      await this.db.updatePledge(pledge._id!, {
        refunded: true,
        refundTxid: txid
      })

      refundTxids.push(txid)
      console.log(`Refunded pledge ${pledge._id} to ${pledge.backerAddress}`)
    }

    // Update campaign status
    await this.db.updateCampaign(campaign._id!, {
      status: 'FAILED',
      updatedAt: new Date()
    })

    return refundTxids
  }

  /**
   * Check if transaction is confirmed
   * In production: Use overlay service
   */
  private async checkTransactionConfirmed(txid: string): Promise<boolean> {
    // TODO: Implement with overlay service or block header tracking
    // For now, assume confirmed after 1 block
    return true
  }
}
```

### Campaign Service

```typescript
// src/services/CampaignService.ts
import { ObjectId } from 'mongodb'
import { Campaign, CampaignStatus, Pledge } from '../models/Campaign'
import { Database } from '../utils/database'
import { EscrowService } from './EscrowService'

export class CampaignService {
  private db: Database
  private escrowService: EscrowService

  constructor(db: Database) {
    this.db = db
    this.escrowService = new EscrowService(db)
  }

  /**
   * Create new crowdfunding campaign
   */
  async createCampaign(params: {
    creatorId: string
    creatorAddress: string
    title: string
    description: string
    goalAmount: number
    durationDays: number
  }): Promise<Campaign> {
    // Generate unique escrow wallet for this campaign
    const { address, privateKey } = await this.escrowService.createEscrowWallet({} as Campaign)

    const deadline = new Date()
    deadline.setDate(deadline.getDate() + params.durationDays)

    const campaign: Campaign = {
      creatorId: params.creatorId,
      creatorAddress: params.creatorAddress,
      title: params.title,
      description: params.description,
      goalAmount: params.goalAmount,
      currentAmount: 0,
      deadline,
      status: CampaignStatus.ACTIVE,
      escrowAddress: address,
      escrowPrivateKey: privateKey, // TODO: Encrypt before storing!
      createdAt: new Date(),
      updatedAt: new Date()
    }

    const result = await this.db.insertCampaign(campaign)
    campaign._id = result.insertedId

    console.log(`Campaign created: ${campaign._id}`)
    return campaign
  }

  /**
   * Record pledge from backer
   */
  async recordPledge(params: {
    campaignId: string
    backerId: string
    backerAddress: string
    amount: number
    txid: string
  }): Promise<Pledge> {
    const campaign = await this.db.getCampaign(params.campaignId)
    if (!campaign) throw new Error('Campaign not found')

    if (campaign.status !== CampaignStatus.ACTIVE) {
      throw new Error('Campaign is not active')
    }

    const pledge: Pledge = {
      campaignId: new ObjectId(params.campaignId),
      backerId: params.backerId,
      backerAddress: params.backerAddress,
      amount: params.amount,
      txid: params.txid,
      refunded: false,
      createdAt: new Date()
    }

    const result = await this.db.insertPledge(pledge)
    pledge._id = result.insertedId

    // Update campaign current amount
    const newAmount = campaign.currentAmount + params.amount
    await this.db.updateCampaign(campaign._id!, {
      currentAmount: newAmount,
      updatedAt: new Date()
    })

    // Check if goal is met
    if (newAmount >= campaign.goalAmount) {
      await this.db.updateCampaign(campaign._id!, {
        status: CampaignStatus.FUNDED
      })
      console.log(`Campaign ${campaign._id} reached goal!`)
    }

    return pledge
  }

  /**
   * Get campaign details
   */
  async getCampaign(campaignId: string): Promise<Campaign | null> {
    return await this.db.getCampaign(campaignId)
  }

  /**
   * Get all campaigns
   */
  async getAllCampaigns(filter?: {
    status?: CampaignStatus
  }): Promise<Campaign[]> {
    return await this.db.getCampaigns(filter)
  }

  /**
   * Get pledges for campaign
   */
  async getCampaignPledges(campaignId: string): Promise<Pledge[]> {
    return await this.db.getPledgesForCampaign(campaignId)
  }
}
```

### Automated Payout Scheduler

```typescript
// src/services/PayoutScheduler.ts
import { CampaignService } from './CampaignService'
import { EscrowService } from './EscrowService'
import { CampaignStatus } from '../models/Campaign'
import { Database } from '../utils/database'

export class PayoutScheduler {
  private campaignService: CampaignService
  private escrowService: EscrowService
  private running: boolean = false

  constructor(db: Database) {
    this.campaignService = new CampaignService(db)
    this.escrowService = new EscrowService(db)
  }

  /**
   * Start automated payout/refund processing
   */
  start() {
    if (this.running) return

    this.running = true
    console.log('Payout scheduler started')

    // Check every 5 minutes
    setInterval(() => this.processScheduledPayouts(), 5 * 60 * 1000)
  }

  /**
   * Process all campaigns that need payouts or refunds
   */
  private async processScheduledPayouts() {
    console.log('Checking for campaigns to process...')

    try {
      // 1. Process successful campaigns
      await this.processFundedCampaigns()

      // 2. Process failed campaigns
      await this.processFailedCampaigns()

    } catch (error) {
      console.error('Error processing payouts:', error)
    }
  }

  /**
   * Process campaigns that reached their goal
   */
  private async processFundedCampaigns() {
    const fundedCampaigns = await this.campaignService.getAllCampaigns({
      status: CampaignStatus.FUNDED
    })

    for (const campaign of fundedCampaigns) {
      try {
        console.log(`Processing payout for campaign ${campaign._id}`)
        const txid = await this.escrowService.processPayout(campaign)
        console.log(`✅ Payout successful: ${txid}`)
      } catch (error) {
        console.error(`Failed to process payout for campaign ${campaign._id}:`, error)
      }
    }
  }

  /**
   * Process campaigns that failed to reach goal by deadline
   */
  private async processFailedCampaigns() {
    const activeCampaigns = await this.campaignService.getAllCampaigns({
      status: CampaignStatus.ACTIVE
    })

    const now = new Date()

    for (const campaign of activeCampaigns) {
      // Check if deadline passed and goal not met
      if (campaign.deadline < now && campaign.currentAmount < campaign.goalAmount) {
        try {
          console.log(`Processing refunds for failed campaign ${campaign._id}`)
          const refundTxids = await this.escrowService.processRefunds(campaign)
          console.log(`✅ Refunds successful: ${refundTxids.length} transactions`)
        } catch (error) {
          console.error(`Failed to process refunds for campaign ${campaign._id}:`, error)
        }
      }
    }
  }

  stop() {
    this.running = false
    console.log('Payout scheduler stopped')
  }
}
```

### API Routes

```typescript
// src/routes/campaigns.ts
import { Router } from 'express'
import { CampaignService } from '../services/CampaignService'
import { Database } from '../utils/database'

export function createCampaignRoutes(db: Database): Router {
  const router = Router()
  const campaignService = new CampaignService(db)

  // Create campaign
  router.post('/campaigns', async (req, res) => {
    try {
      const campaign = await campaignService.createCampaign(req.body)
      res.json({ success: true, campaign })
    } catch (error: any) {
      res.status(400).json({ success: false, error: error.message })
    }
  })

  // Get all campaigns
  router.get('/campaigns', async (req, res) => {
    try {
      const campaigns = await campaignService.getAllCampaigns()
      res.json({ success: true, campaigns })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Get single campaign
  router.get('/campaigns/:id', async (req, res) => {
    try {
      const campaign = await campaignService.getCampaign(req.params.id)
      if (!campaign) {
        return res.status(404).json({ success: false, error: 'Campaign not found' })
      }
      res.json({ success: true, campaign })
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message })
    }
  })

  // Record pledge
  router.post('/campaigns/:id/pledge', async (req, res) => {
    try {
      const pledge = await campaignService.recordPledge({
        campaignId: req.params.id,
        ...req.body
      })
      res.json({ success: true, pledge })
    } catch (error: any) {
      res.status(400).json({ success: false, error: error.message })
    }
  })

  // Get campaign pledges
  router.get('/campaigns/:id/pledges', async (req, res) => {
    try {
      const pledges = await campaignService.getCampaignPledges(req.params.id)
      res.json({ success: true, pledges })
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
import { createCampaignRoutes } from './routes/campaigns'
import { PayoutScheduler } from './services/PayoutScheduler'

dotenv.config()

async function main() {
  const app = express()
  app.use(express.json())

  // Connect to database
  const db = new Database(process.env.MONGODB_URI!)
  await db.connect()

  // Setup routes
  app.use('/api', createCampaignRoutes(db))

  // Start payout scheduler
  const scheduler = new PayoutScheduler(db)
  scheduler.start()

  // Start server
  const PORT = process.env.PORT || 3000
  app.listen(PORT, () => {
    console.log(`🚀 Crowdfunding Platform API running on port ${PORT}`)
  })
}

main().catch(console.error)
```

---

## Part 2: Frontend Implementation

### React Components

#### Campaign Dashboard

```typescript
// src/components/CampaignDashboard.tsx
import React, { useState, useEffect } from 'react'
import { Campaign } from '../types'
import { CampaignCard } from './CampaignCard'
import { useWallet } from '../hooks/useWallet'

export const CampaignDashboard: React.FC = () => {
  const [campaigns, setCampaigns] = useState<Campaign[]>([])
  const [loading, setLoading] = useState(true)
  const { connected } = useWallet()

  useEffect(() => {
    loadCampaigns()
  }, [])

  const loadCampaigns = async () => {
    try {
      const response = await fetch('/api/campaigns')
      const data = await response.json()
      setCampaigns(data.campaigns)
    } catch (error) {
      console.error('Failed to load campaigns:', error)
    } finally {
      setLoading(false)
    }
  }

  if (loading) return <div>Loading campaigns...</div>

  return (
    <div className="campaign-dashboard">
      <h1>Active Campaigns</h1>

      {!connected && (
        <div className="connect-prompt">
          Connect your wallet to back campaigns
        </div>
      )}

      <div className="campaign-grid">
        {campaigns.map(campaign => (
          <CampaignCard
            key={campaign._id}
            campaign={campaign}
            onPledge={loadCampaigns}
          />
        ))}
      </div>
    </div>
  )
}
```

#### Campaign Card with Pledge

```typescript
// src/components/CampaignCard.tsx
import React, { useState } from 'react'
import { WalletClient } from '@bsv/sdk'
import { Campaign } from '../types'
import { useWallet } from '../hooks/useWallet'

interface Props {
  campaign: Campaign
  onPledge: () => void
}

export const CampaignCard: React.FC<Props> = ({ campaign, onPledge }) => {
  const { wallet, address } = useWallet()
  const [pledging, setPledging] = useState(false)
  const [pledgeAmount, setPledgeAmount] = useState('')

  const progressPercent = (campaign.currentAmount / campaign.goalAmount) * 100

  const handlePledge = async () => {
    if (!wallet || !address) {
      alert('Please connect your wallet')
      return
    }

    setPledging(true)

    try {
      const amountSats = parseInt(pledgeAmount)

      // Send pledge to campaign escrow address via WalletClient
      const result = await wallet.createAction({
        description: `Pledge to ${campaign.title}`,
        outputs: [{
          lockingScript: new P2PKH().lock(campaign.escrowAddress).toHex(),
          satoshis: amountSats,
          outputDescription: 'Campaign pledge'
        }]
      })

      // Record pledge in backend
      await fetch(`/api/campaigns/${campaign._id}/pledge`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          backerId: address,
          backerAddress: address,
          amount: amountSats,
          txid: result.txid
        })
      })

      alert(`Pledge successful! TXID: ${result.txid}`)
      setPledgeAmount('')
      onPledge()

    } catch (error: any) {
      console.error('Pledge failed:', error)
      alert(error.message)
    } finally {
      setPledging(false)
    }
  }

  return (
    <div className="campaign-card">
      <h3>{campaign.title}</h3>
      <p>{campaign.description}</p>

      <div className="progress-bar">
        <div className="progress" style={{ width: `${progressPercent}%` }} />
      </div>

      <div className="campaign-stats">
        <div>
          <strong>{campaign.currentAmount.toLocaleString()}</strong> sats raised
        </div>
        <div>
          Goal: <strong>{campaign.goalAmount.toLocaleString()}</strong> sats
        </div>
        <div>
          {progressPercent.toFixed(1)}% funded
        </div>
      </div>

      <div className="deadline">
        Deadline: {new Date(campaign.deadline).toLocaleDateString()}
      </div>

      {campaign.status === 'ACTIVE' && (
        <div className="pledge-form">
          <input
            type="number"
            placeholder="Amount (satoshis)"
            value={pledgeAmount}
            onChange={(e) => setPledgeAmount(e.target.value)}
            disabled={pledging}
          />
          <button
            onClick={handlePledge}
            disabled={pledging || !pledgeAmount}
          >
            {pledging ? 'Processing...' : 'Back This Campaign'}
          </button>
        </div>
      )}

      {campaign.status === 'FUNDED' && (
        <div className="status funded">✅ Funded - Payout in progress</div>
      )}

      {campaign.status === 'PAID_OUT' && (
        <div className="status success">✅ Successfully Funded</div>
      )}

      {campaign.status === 'FAILED' && (
        <div className="status failed">❌ Campaign Failed - Refunds processed</div>
      )}
    </div>
  )
}
```

---

## Testing

### Test Campaign Creation

```typescript
// tests/create-campaign.test.ts
import { CampaignService } from '../src/services/CampaignService'

describe('Campaign Creation', () => {
  it('should create campaign with unique escrow address', async () => {
    const service = new CampaignService(db)

    const campaign = await service.createCampaign({
      creatorId: 'user123',
      creatorAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
      title: 'Test Campaign',
      description: 'Testing crowdfunding',
      goalAmount: 100000, // 0.001 BSV
      durationDays: 30
    })

    expect(campaign.escrowAddress).toBeDefined()
    expect(campaign.status).toBe('ACTIVE')
    expect(campaign.currentAmount).toBe(0)
  })
})
```

### Test Payout Flow

```typescript
// tests/payout.test.ts
describe('Payout Processing', () => {
  it('should process payout when goal is met', async () => {
    // Create campaign
    // Record pledges totaling goal amount
    // Trigger payout
    // Verify creator received funds
    // Verify campaign status updated
  })
})
```

---

## Deployment

### Environment Variables

```env
# Network
NETWORK=testnet

# Database
MONGODB_URI=mongodb://localhost:27017/crowdfunding

# Platform
PLATFORM_ADDRESS=your-platform-address
PLATFORM_FEE_PERCENT=5

# API
PORT=3000
```

### Production Considerations

**Security:**
- ✅ Encrypt escrow private keys in database
- ✅ Use HSM for escrow key storage
- ✅ Rate limiting on API endpoints
- ✅ Input validation and sanitization

**Scalability:**
- ✅ Use overlay service for transaction monitoring
- ✅ Implement queueing for payout processing
- ✅ Cache campaign data
- ✅ Optimize database queries

**Monitoring:**
- ✅ Track failed payouts/refunds
- ✅ Alert on stuck transactions
- ✅ Dashboard for platform metrics

---

## Summary

You've built a complete crowdfunding platform demonstrating:

✅ **Escrow patterns** - Securely holding funds
✅ **Automated payouts** - Processing when conditions met
✅ **Refund logic** - Returning funds when campaigns fail
✅ **WalletClient integration** - User-controlled pledges
✅ **Backend services** - Campaign management and automation
✅ **Real-world BSV application** - Production-ready patterns

---

## Next Steps

- **Enhance**: Add campaign updates, backer rewards, stretch goals
- **Scale**: Implement overlay service integration
- **Deploy**: Host on production infrastructure
- **Continue**: [Project 2: Asset Tokenization](../asset-tokenization/)

---

## Resources

- [SDK Transaction Documentation](../../../sdk-components/transaction/)
- [WalletClient Guide](../../beginner/wallet-client-integration/)
- [Escrow Patterns](../../../code-features/smart-contracts/)
- [Multi-Output Transactions](../../../code-features/payment-distribution/)
