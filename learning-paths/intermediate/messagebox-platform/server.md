# MessageBox Platform - Server Guide

**Building Server-Side MessageBox Applications with BSV SDK**

This guide covers server-side patterns for implementing MessageBox functionality using Node.js/Express, including wallet session management, certification endpoints, and database storage.

> **📦 Complete Code Repository**: [https://github.com/bsv-blockchain-demos/messagebox-platform](https://github.com/bsv-blockchain-demos/messagebox-platform)
>
> All code examples in this guide are taken from the working implementation in the `server/` directory.

---

## Table of Contents

1. [Setup](#setup)
2. [Session Management](#session-management)
3. [Certification Endpoint](#certification-endpoint)
4. [Database Storage](#database-storage)
5. [Payment Endpoints](#payment-endpoints)
6. [Complete Server](#complete-server)

---

## Setup

### Installation

```bash
npm install @bsv/sdk @bsv/message-box-client express mongodb dotenv
npm install -D @types/node @types/express typescript
```

### Project Structure

```
server/
├── src/
│   ├── index.ts                    # Main server
│   ├── wallet/
│   │   └── WalletSessionManager.ts # Session management
│   ├── storage/
│   │   └── CertificationStorage.ts # MongoDB storage
│   └── routes/
│       ├── wallet.ts               # Wallet endpoints
│       ├── certify.ts              # Certification endpoints
│       └── payment.ts              # Payment endpoints
├── .env
├── package.json
└── tsconfig.json
```

### Environment Configuration

Create `.env`:

```env
PORT=3000
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB=messagebox_platform
MESSAGEBOX_HOST=https://messagebox.babbage.systems
```

### TypeScript Configuration

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

---

## Session Management

### Wallet Session Manager

Manages wallet sessions with 30-minute timeout and automatic cleanup:

> **📁 See full implementation**: [`server/src/wallet/WalletSessionManager.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/server/src/wallet/WalletSessionManager.ts)

```typescript
// src/wallet/WalletSessionManager.ts

import crypto from 'crypto'

interface WalletSession {
  sessionId: string
  walletClient: any | null
  identityKey: string
  createdAt: Date
  lastAccessedAt: Date
}

/**
 * Manages wallet sessions with automatic expiration
 * Singleton pattern ensures one instance across server
 */
export class WalletSessionManager {
  private static instance: WalletSessionManager
  private sessions: Map<string, WalletSession>
  private readonly SESSION_TIMEOUT = 30 * 60 * 1000 // 30 minutes
  private cleanupInterval: NodeJS.Timeout

  private constructor() {
    this.sessions = new Map()

    // Auto-cleanup expired sessions every 5 minutes
    this.cleanupInterval = setInterval(() => {
      this.cleanupExpiredSessions()
    }, 5 * 60 * 1000)
  }

  public static getInstance(): WalletSessionManager {
    if (!WalletSessionManager.instance) {
      WalletSessionManager.instance = new WalletSessionManager()
    }
    return WalletSessionManager.instance
  }

  /**
   * Create new session with identity key
   */
  public createSessionWithIdentity(identityKey: string): string {
    const sessionId = crypto.randomBytes(32).toString('hex')

    this.sessions.set(sessionId, {
      sessionId,
      walletClient: null,
      identityKey,
      createdAt: new Date(),
      lastAccessedAt: new Date()
    })

    console.log(`✓ Session created: ${sessionId.substring(0, 16)}... for ${identityKey.substring(0, 20)}...`)
    return sessionId
  }

  /**
   * Get wallet client for session
   * Updates lastAccessedAt to keep session alive
   */
  public getWalletClient(sessionId: string): any | null {
    const session = this.sessions.get(sessionId)

    if (!session) {
      return null
    }

    // Check if expired
    const now = new Date()
    const timeSinceAccess = now.getTime() - session.lastAccessedAt.getTime()

    if (timeSinceAccess > this.SESSION_TIMEOUT) {
      console.log(`Session expired: ${sessionId.substring(0, 16)}...`)
      this.sessions.delete(sessionId)
      return null
    }

    // Update last accessed time
    session.lastAccessedAt = new Date()

    return session.walletClient
  }

  /**
   * Get session info (identity key, created time)
   */
  public getSessionInfo(sessionId: string): {
    identityKey: string
    createdAt: Date
  } | null {
    const session = this.sessions.get(sessionId)

    if (!session) {
      return null
    }

    // Check if expired
    const now = new Date()
    const timeSinceAccess = now.getTime() - session.lastAccessedAt.getTime()

    if (timeSinceAccess > this.SESSION_TIMEOUT) {
      this.sessions.delete(sessionId)
      return null
    }

    // Update last accessed time
    session.lastAccessedAt = new Date()

    return {
      identityKey: session.identityKey,
      createdAt: session.createdAt
    }
  }

  /**
   * Delete session (logout)
   */
  public deleteSession(sessionId: string): boolean {
    const deleted = this.sessions.delete(sessionId)
    if (deleted) {
      console.log(`✓ Session deleted: ${sessionId.substring(0, 16)}...`)
    }
    return deleted
  }

  /**
   * Cleanup expired sessions
   */
  private cleanupExpiredSessions(): void {
    const now = new Date()
    let cleanedCount = 0

    for (const [sessionId, session] of this.sessions.entries()) {
      const timeSinceAccess = now.getTime() - session.lastAccessedAt.getTime()

      if (timeSinceAccess > this.SESSION_TIMEOUT) {
        this.sessions.delete(sessionId)
        cleanedCount++
      }
    }

    if (cleanedCount > 0) {
      console.log(`✓ Cleaned up ${cleanedCount} expired session(s)`)
    }
  }

  /**
   * Get active session count
   */
  public getSessionCount(): number {
    return this.sessions.size
  }

  /**
   * Stop cleanup interval (for graceful shutdown)
   */
  public destroy(): void {
    clearInterval(this.cleanupInterval)
  }
}
```

**Key Features:**
- 30-minute session timeout
- Auto-cleanup every 5 minutes
- Session refresh on each access
- Singleton pattern for global state

---

## Database Storage

### MongoDB Certification Storage

Store and query certified user identities:

> **📁 See full implementation**: [`server/src/storage/CertificationStorage.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/server/src/storage/CertificationStorage.ts)

```typescript
// src/storage/CertificationStorage.ts

import { MongoClient, Db, Collection } from 'mongodb'

export interface CertifiedUser {
  identityKey: string
  alias?: string
  certificationDate: Date
  certificationTxid: string
  host: string
}

/**
 * MongoDB storage for certified users
 * Singleton pattern
 */
export class CertificationStorage {
  private static instance: CertificationStorage
  private client: MongoClient | null = null
  private db: Db | null = null
  private collection: Collection<CertifiedUser> | null = null

  private constructor() {}

  public static getInstance(): CertificationStorage {
    if (!CertificationStorage.instance) {
      CertificationStorage.instance = new CertificationStorage()
    }
    return CertificationStorage.instance
  }

  /**
   * Connect to MongoDB
   */
  public async connect(uri: string, dbName: string): Promise<void> {
    try {
      this.client = new MongoClient(uri)
      await this.client.connect()

      this.db = this.client.db(dbName)
      this.collection = this.db.collection<CertifiedUser>('certified_users')

      // Create unique index on identityKey
      await this.collection.createIndex({ identityKey: 1 }, { unique: true })

      // Create text index on alias for search
      await this.collection.createIndex({ alias: 'text' })

      console.log('✓ MongoDB connected:', dbName)
    } catch (error) {
      console.error('MongoDB connection failed:', error)
      throw error
    }
  }

  /**
   * Store or update certification
   */
  public async storeCertification(user: {
    identityKey: string
    alias?: string
    txid: string
    host: string
  }): Promise<void> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    await this.collection.updateOne(
      { identityKey: user.identityKey },
      {
        $set: {
          identityKey: user.identityKey,
          alias: user.alias,
          certificationTxid: user.txid,
          host: user.host,
          certificationDate: new Date()
        }
      },
      { upsert: true }
    )

    console.log(`✓ Stored certification for: ${user.identityKey.substring(0, 20)}...`)
  }

  /**
   * Get certified user by identity key
   */
  public async getCertifiedUser(identityKey: string): Promise<CertifiedUser | null> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    return await this.collection.findOne({ identityKey })
  }

  /**
   * List all certified users
   */
  public async listAllCertifiedUsers(): Promise<CertifiedUser[]> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    return await this.collection
      .find({})
      .sort({ certificationDate: -1 })
      .toArray()
  }

  /**
   * Search users by alias (case-insensitive)
   */
  public async searchByAlias(query: string): Promise<CertifiedUser[]> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    return await this.collection
      .find({
        alias: { $regex: query, $options: 'i' }
      })
      .sort({ certificationDate: -1 })
      .toArray()
  }

  /**
   * Close connection
   */
  public async disconnect(): Promise<void> {
    if (this.client) {
      await this.client.close()
      console.log('✓ MongoDB disconnected')
    }
  }
}
```

**Database Schema:**

```typescript
{
  identityKey: "03abc123...",        // Unique identifier
  alias: "Alice",                    // Optional human-readable name
  certificationDate: Date,           // When certified
  certificationTxid: "abc123...",    // Blockchain proof
  host: "messagebox.babbage.systems" // MessageBox host
}
```

---

## Certification Endpoint

### Store Certification Route

> **📁 See full implementation**: [`server/src/routes/certify.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/server/src/routes/certify.ts)

```typescript
// src/routes/certify.ts

import { Router } from 'express'
import { CertificationStorage } from '../storage/CertificationStorage'

const router = Router()

/**
 * POST /api/store-certification
 * Store certification data from frontend
 */
router.post('/store-certification', async (req, res) => {
  try {
    const { identityKey, alias, txid } = req.body

    if (!identityKey || !txid) {
      return res.status(400).json({
        success: false,
        error: 'identityKey and txid are required'
      })
    }

    const storage = CertificationStorage.getInstance()
    const host = process.env.MESSAGEBOX_HOST || 'https://messagebox.babbage.systems'

    await storage.storeCertification({
      identityKey,
      alias,
      txid,
      host
    })

    res.json({
      success: true,
      message: 'Certification stored successfully'
    })
  } catch (error: any) {
    console.error('Store certification error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

/**
 * GET /api/certified-users
 * Get all certified users
 */
router.get('/certified-users', async (req, res) => {
  try {
    const storage = CertificationStorage.getInstance()
    const users = await storage.listAllCertifiedUsers()

    res.json({
      success: true,
      users
    })
  } catch (error: any) {
    console.error('List users error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

/**
 * GET /api/certified-users/search?q=query
 * Search users by alias
 */
router.get('/certified-users/search', async (req, res) => {
  try {
    const query = req.query.q as string

    if (!query) {
      return res.status(400).json({
        success: false,
        error: 'Query parameter "q" is required'
      })
    }

    const storage = CertificationStorage.getInstance()
    const users = await storage.searchByAlias(query)

    res.json({
      success: true,
      users
    })
  } catch (error: any) {
    console.error('Search users error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

export default router
```

---

## Wallet Endpoints

### Wallet Connection Routes

> **📁 See full implementation**: [`server/src/routes/wallet.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/server/src/routes/wallet.ts)

```typescript
// src/routes/wallet.ts

import { Router } from 'express'
import { WalletSessionManager } from '../wallet/WalletSessionManager'

const router = Router()

/**
 * POST /api/wallet/connect
 * Create session for connected wallet
 */
router.post('/wallet/connect', async (req, res) => {
  try {
    const { identityKey } = req.body

    if (!identityKey) {
      return res.status(400).json({
        success: false,
        error: 'identityKey is required'
      })
    }

    const sessionManager = WalletSessionManager.getInstance()
    const sessionId = sessionManager.createSessionWithIdentity(identityKey)

    res.json({
      success: true,
      sessionId,
      identityKey,
      message: 'Wallet connected successfully'
    })
  } catch (error: any) {
    console.error('Connect wallet error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

/**
 * GET /api/wallet/status
 * Check session status
 */
router.get('/wallet/status', async (req, res) => {
  try {
    const sessionId = req.headers['x-session-id'] as string

    if (!sessionId) {
      return res.status(401).json({
        success: false,
        connected: false,
        error: 'Session ID required'
      })
    }

    const sessionManager = WalletSessionManager.getInstance()
    const sessionInfo = sessionManager.getSessionInfo(sessionId)

    if (!sessionInfo) {
      return res.status(401).json({
        success: false,
        connected: false,
        error: 'Invalid or expired session'
      })
    }

    res.json({
      success: true,
      connected: true,
      identityKey: sessionInfo.identityKey,
      connectedAt: sessionInfo.createdAt
    })
  } catch (error: any) {
    console.error('Status check error:', error)
    res.status(500).json({
      success: false,
      connected: false,
      error: error.message
    })
  }
})

/**
 * POST /api/wallet/disconnect
 * Delete session
 */
router.post('/wallet/disconnect', async (req, res) => {
  try {
    const sessionId = req.headers['x-session-id'] as string

    if (!sessionId) {
      return res.status(400).json({
        success: false,
        error: 'Session ID required'
      })
    }

    const sessionManager = WalletSessionManager.getInstance()
    const deleted = sessionManager.deleteSession(sessionId)

    res.json({
      success: true,
      disconnected: deleted,
      message: deleted ? 'Session deleted' : 'Session not found'
    })
  } catch (error: any) {
    console.error('Disconnect error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

export default router
```

---

## Payment Endpoints

### Payment Initiation Route (Optional)

> **📁 See full implementation**: [`server/src/routes/payment.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/server/src/routes/payment.ts)

```typescript
// src/routes/payment.ts

import { Router } from 'express'
import { WalletSessionManager } from '../wallet/WalletSessionManager'
import { CertificationStorage } from '../storage/CertificationStorage'

const router = Router()

/**
 * POST /api/initiate-payment
 * Validate payment recipient
 * Note: Actual payment happens on frontend with WalletClient
 */
router.post('/initiate-payment', async (req, res) => {
  try {
    const { recipient, amount } = req.body
    const sessionId = req.headers['x-session-id'] as string

    // Validate input
    if (!recipient || !amount) {
      return res.status(400).json({
        success: false,
        error: 'Recipient and amount are required'
      })
    }

    if (!sessionId) {
      return res.status(401).json({
        success: false,
        error: 'Session ID required'
      })
    }

    // Verify session exists
    const sessionManager = WalletSessionManager.getInstance()
    const sessionInfo = sessionManager.getSessionInfo(sessionId)

    if (!sessionInfo) {
      return res.status(401).json({
        success: false,
        error: 'Invalid or expired session'
      })
    }

    // Verify recipient is certified
    const storage = CertificationStorage.getInstance()
    const recipientUser = await storage.getCertifiedUser(recipient)

    if (!recipientUser) {
      return res.status(404).json({
        success: false,
        error: 'Recipient not found or not certified'
      })
    }

    res.json({
      success: true,
      recipient: recipientUser,
      message: 'Recipient verified, proceed with payment'
    })
  } catch (error: any) {
    console.error('Payment initiation error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

export default router
```

**Note:** This endpoint only validates the recipient. Actual payment transactions are created and broadcast on the frontend using WalletClient and PeerPayClient for security.

---

## Complete Server

### Main Server Setup

> **📁 See full implementation**: [`server/src/index.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/server/src/index.ts)

```typescript
// src/index.ts

import express from 'express'
import cors from 'cors'
import { config } from 'dotenv'
import { CertificationStorage } from './storage/CertificationStorage'
import { WalletSessionManager } from './wallet/WalletSessionManager'
import walletRoutes from './routes/wallet'
import certifyRoutes from './routes/certify'
import paymentRoutes from './routes/payment'

// Load environment variables
config()

const app = express()
const PORT = parseInt(process.env.PORT || '3000')

// Middleware
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3001',
  credentials: true
}))
app.use(express.json())

// Request logging
app.use((req, res, next) => {
  console.log(`${req.method} ${req.path}`)
  next()
})

// Routes
app.use('/api', walletRoutes)
app.use('/api', certifyRoutes)
app.use('/api', paymentRoutes)

// Health check
app.get('/health', (req, res) => {
  const sessionManager = WalletSessionManager.getInstance()
  res.json({
    status: 'ok',
    activeSessions: sessionManager.getSessionCount(),
    timestamp: new Date().toISOString()
  })
})

// Error handler
app.use((err: any, req: express.Request, res: express.Response, next: express.NextFunction) => {
  console.error('Server error:', err)
  res.status(500).json({
    success: false,
    error: err.message || 'Internal server error'
  })
})

// Start server
async function main() {
  try {
    // Connect to MongoDB
    const storage = CertificationStorage.getInstance()
    await storage.connect(
      process.env.MONGODB_URI!,
      process.env.MONGODB_DB || 'messagebox_platform'
    )

    // Start server
    app.listen(PORT, () => {
      console.log('\n✅ MessageBox Server Running')
      console.log(`📡 API: http://localhost:${PORT}`)
      console.log(`🏥 Health: http://localhost:${PORT}/health`)
      console.log('\nEndpoints:')
      console.log('  POST   /api/wallet/connect')
      console.log('  GET    /api/wallet/status')
      console.log('  POST   /api/wallet/disconnect')
      console.log('  POST   /api/store-certification')
      console.log('  GET    /api/certified-users')
      console.log('  GET    /api/certified-users/search?q=query')
      console.log('  POST   /api/initiate-payment')
    })

    // Graceful shutdown
    process.on('SIGTERM', async () => {
      console.log('\nShutting down gracefully...')
      const sessionManager = WalletSessionManager.getInstance()
      sessionManager.destroy()
      await storage.disconnect()
      process.exit(0)
    })
  } catch (error) {
    console.error('Failed to start server:', error)
    process.exit(1)
  }
}

main()
```

---

## Testing

### Start MongoDB

```bash
# Using Docker
docker run -d -p 27017:27017 --name mongodb mongo:latest

# Or install locally
mongod --dbpath ./data
```

### Start Server

```bash
npm run build
npm start
```

### Test Endpoints

```bash
# Health check
curl http://localhost:3000/health

# Connect wallet (from frontend)
curl -X POST http://localhost:3000/api/wallet/connect \
  -H "Content-Type: application/json" \
  -d '{"identityKey":"03abc123..."}'

# Store certification
curl -X POST http://localhost:3000/api/store-certification \
  -H "Content-Type: application/json" \
  -d '{"identityKey":"03abc123...","alias":"Alice","txid":"abc123..."}'

# List certified users
curl http://localhost:3000/api/certified-users

# Search users
curl "http://localhost:3000/api/certified-users/search?q=Alice"
```

---

## Deployment Considerations

### Environment Variables

Production `.env`:

```env
NODE_ENV=production
PORT=3000
MONGODB_URI=mongodb://username:password@host:27017/dbname?authSource=admin
MONGODB_DB=messagebox_platform
MESSAGEBOX_HOST=https://messagebox.babbage.systems
FRONTEND_URL=https://your-frontend.com
```

### Security Best Practices

1. **HTTPS Only**: Use SSL/TLS certificates
2. **Rate Limiting**: Prevent abuse with rate limits
3. **Input Validation**: Validate all user inputs
4. **CORS**: Restrict origins in production
5. **Session Secrets**: Use strong random session IDs
6. **Database Credentials**: Secure MongoDB with authentication

### Scaling Considerations

1. **Session Storage**: Use Redis for distributed sessions
2. **Database Replication**: MongoDB replica sets for redundancy
3. **Load Balancing**: Distribute requests across multiple servers
4. **Caching**: Cache certified users list with TTL
5. **Monitoring**: Add logging and metrics

---

## Summary

This server guide covered:

- **Session Management** - Managing wallet sessions with expiration
- **Database Storage** - MongoDB for certified user storage
- **API Endpoints** - RESTful routes for wallet, certification, and payments
- **Security Patterns** - Session validation and authentication
- **Deployment** - Production considerations and best practices

These patterns provide a production-ready backend for MessageBox applications.

---

## Next Steps

- [Frontend Implementation Guide](./frontend.md) - Learn browser-side patterns
- [Overlay Services](../overlay-services/README.md) - Deploy custom MessageBox infrastructure
- [MongoDB Documentation](https://docs.mongodb.com/) - Database optimization
- [Express Best Practices](https://expressjs.com/en/advanced/best-practice-security.html) - Security hardening
