# Development Environment Setup

**Module 1: Setting Up Your BSV Development Environment**

This module guides you through setting up your development environment for BSV blockchain development. The setup differs depending on whether you're building backend services or frontend dApps.

---

## Universal Prerequisites

**Required for both paradigms:**

- Computer running Windows, macOS, or Linux
- Basic command line knowledge
- Text editor or IDE (VS Code recommended)
- Node.js 18+ and npm

---

## Part 1: Universal Setup (Both Paradigms)

### Step 1: Install Node.js

The BSV TypeScript SDK requires Node.js version 18 or higher.

#### macOS
```bash
# Using Homebrew
brew install node

# Verify installation
node --version  # Should be v18.0.0 or higher
npm --version
```

#### Windows
1. Download installer from [nodejs.org](https://nodejs.org/) (LTS version)
2. Run the installer
3. Verify in Command Prompt:
```cmd
node --version
npm --version
```

#### Linux (Ubuntu/Debian)
```bash
# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installation
node --version
npm --version
```

### Step 2: Install TypeScript

```bash
# Install TypeScript globally
npm install -g typescript

# Verify installation
tsc --version  # Should be 5.0.0 or higher
```

### Step 3: Set Up Your IDE

We recommend **Visual Studio Code** for TypeScript development:

1. Download from [code.visualstudio.com](https://code.visualstudio.com/)
2. Install these essential extensions:
   - **ESLint**: Code quality and linting
   - **Prettier**: Automatic code formatting
   - **TypeScript Import Sorter**: Organize imports
   - **GitLens**: Git integration and history

#### VS Code Settings

Create `.vscode/settings.json` in your project:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "typescript.preferences.importModuleSpecifier": "relative",
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  }
}
```

### Step 4: Install BSV SDK

```bash
# Create project directory
mkdir my-bsv-app
cd my-bsv-app

# Initialize npm project
npm init -y

# Install BSV SDK (core dependency)
npm install @bsv/sdk

# Install development dependencies
npm install -D typescript @types/node ts-node nodemon
```

### Step 5: Configure TypeScript

Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Step 6: Set Up Git (Recommended)

```bash
# Initialize git repository
git init

# Create .gitignore
cat > .gitignore << 'EOF'
node_modules/
dist/
.env
.env.local
*.log
.DS_Store
.vscode/
.idea/
EOF

# First commit
git add .
git commit -m "Initial BSV project setup"
```

---

## Part 2A: Backend/Service Development Setup

**For custodial applications where you control private keys**

### Backend Project Structure

```bash
# Create backend directories
mkdir -p src/config
mkdir -p src/services
mkdir -p src/models
mkdir -p src/utils
mkdir -p src/api
```

```
my-bsv-backend/
├── src/
│   ├── index.ts              # Main server entry point
│   ├── config/
│   │   ├── database.ts       # Database configuration
│   │   └── wallet.ts         # Wallet configuration
│   ├── services/
│   │   ├── wallet.service.ts # Wallet management logic
│   │   └── payment.service.ts# Payment processing
│   ├── models/
│   │   ├── wallet.model.ts   # Wallet data model
│   │   └── transaction.model.ts
│   ├── utils/
│   │   └── crypto.ts         # Cryptographic utilities
│   └── api/
│       └── routes.ts         # API endpoints
├── dist/                     # Compiled output (gitignored)
├── node_modules/             # Dependencies (gitignored)
├── .env                      # Environment variables (gitignored)
├── .gitignore
├── package.json
├── tsconfig.json
└── README.md
```

### Install Backend Dependencies

```bash
# Install additional backend dependencies
npm install express dotenv
npm install -D @types/express @types/dotenv
```

### Environment Variables (Backend)

Create `.env` file:
```env
# NEVER COMMIT THIS FILE!

# Network
NETWORK=testnet  # or mainnet

# Database (example with MongoDB)
DATABASE_URL=mongodb://localhost:27017/bsv-app

# API Configuration
PORT=3000
API_SECRET=your-secret-key-here

# Security
ENCRYPTION_KEY=your-encryption-key-here

# Note: Private keys should be stored in secure vault/HSM in production
# For development only:
# MASTER_PRIVATE_KEY_WIF=cVrtH...
```

**⚠️ Security Warning:**
- Never store production private keys in `.env` files
- Use HSM (Hardware Security Module) or key vaults in production
- Always encrypt private keys at rest
- Use strong access controls

### Backend Test File

Create `src/index.ts`:
```typescript
import { PrivateKey, Transaction, P2PKH } from '@bsv/sdk'
import * as dotenv from 'dotenv'

dotenv.config()

async function testBackendSetup() {
  console.log('=== BSV Backend Development Environment Test ===\n')

  // Generate a private key (in production, load from secure storage)
  const privateKey = PrivateKey.fromRandom()
  const address = privateKey.toPublicKey().toAddress()

  console.log('✅ Private key generated')
  console.log('Address:', address)
  console.log('Network:', process.env.NETWORK || 'testnet')

  // Test transaction creation (won't broadcast without UTXOs)
  const tx = new Transaction()

  console.log('\n✅ Transaction object created')
  console.log('✅ BSV SDK is working correctly!')
  console.log('\n🚀 Ready for backend development')
}

testBackendSetup().catch(console.error)
```

### Backend NPM Scripts

Update `package.json`:
```json
{
  "name": "my-bsv-backend",
  "version": "1.0.0",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon --watch src --exec ts-node src/index.ts",
    "build": "tsc",
    "test": "ts-node src/index.ts"
  },
  "dependencies": {
    "@bsv/sdk": "^1.0.0",
    "express": "^4.18.0",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.0",
    "@types/dotenv": "^8.2.0",
    "ts-node": "^10.9.0",
    "nodemon": "^3.0.0",
    "typescript": "^5.0.0"
  }
}
```

### Test Backend Setup

```bash
npm run test

# Expected output:
# === BSV Backend Development Environment Test ===
# ✅ Private key generated
# Address: 1A1zP1...
# Network: testnet
# ✅ Transaction object created
# ✅ BSV SDK is working correctly!
# 🚀 Ready for backend development
```

---

## Part 2B: Frontend/dApp Development Setup

**For non-custodial applications using WalletClient**

### Frontend Project Structure (React Example)

```bash
# Create React app with TypeScript
npx create-react-app my-bsv-dapp --template typescript

cd my-bsv-dapp

# Install BSV SDK
npm install @bsv/sdk
```

```
my-bsv-dapp/
├── public/
│   └── index.html
├── src/
│   ├── components/
│   │   ├── WalletConnect.tsx    # Wallet connection UI
│   │   ├── SendPayment.tsx      # Payment component
│   │   └── TransactionList.tsx  # Transaction history
│   ├── services/
│   │   └── wallet.service.ts    # WalletClient wrapper
│   ├── hooks/
│   │   └── useWallet.ts         # React hook for wallet
│   ├── App.tsx                  # Main app component
│   └── index.tsx                # Entry point
├── node_modules/
├── .gitignore
├── package.json
├── tsconfig.json
└── README.md
```

### Install Frontend Dependencies

```bash
# No additional BSV dependencies needed!
# WalletClient is included in @bsv/sdk

# Optional: UI library
npm install @mui/material @emotion/react @emotion/styled
```

### Frontend Test Component

Create `src/components/WalletTest.tsx`:
```typescript
import React, { useState } from 'react'
import { WalletClient } from '@bsv/sdk'

export const WalletTest: React.FC = () => {
  const [wallet, setWallet] = useState<WalletClient | null>(null)
  const [address, setAddress] = useState<string>('')
  const [status, setStatus] = useState<string>('Not connected')

  const connectWallet = async () => {
    try {
      setStatus('Connecting...')

      // Initialize WalletClient
      const walletClient = new WalletClient()

      // Connect to MetaNet Desktop Wallet
      await walletClient.connect()

      // Get user's address
      const userAddress = await walletClient.getAddress()

      setWallet(walletClient)
      setAddress(userAddress)
      setStatus('Connected')

      console.log('✅ Wallet connected:', userAddress)
    } catch (error: any) {
      console.error('❌ Connection failed:', error)
      setStatus(`Error: ${error.message}`)
    }
  }

  return (
    <div style={{ padding: '20px' }}>
      <h2>BSV Wallet Connection Test</h2>

      <div>
        <p><strong>Status:</strong> {status}</p>
        {address && <p><strong>Address:</strong> {address}</p>}
      </div>

      {!wallet ? (
        <button onClick={connectWallet}>
          Connect MetaNet Desktop Wallet
        </button>
      ) : (
        <div>
          <p>✅ Wallet connected successfully!</p>
          <p>Your dApp can now request transactions from the user's wallet.</p>
        </div>
      )}

      <div style={{ marginTop: '20px', padding: '10px', background: '#f0f0f0' }}>
        <p><strong>Prerequisites:</strong></p>
        <ul>
          <li>MetaNet Desktop Wallet installed</li>
          <li>Wallet must be unlocked</li>
          <li>Your dApp origin must be authorized</li>
        </ul>
      </div>
    </div>
  )
}
```

### Update App.tsx

```typescript
import React from 'react'
import { WalletTest } from './components/WalletTest'
import './App.css'

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <h1>BSV dApp Development Environment</h1>
        <WalletTest />
      </header>
    </div>
  )
}

export default App
```

### Frontend NPM Scripts

`package.json` (already configured by Create React App):
```json
{
  "name": "my-bsv-dapp",
  "version": "0.1.0",
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "dependencies": {
    "@bsv/sdk": "^1.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.0.0"
  }
}
```

### Test Frontend Setup

```bash
npm start

# Browser opens at http://localhost:3000
# Click "Connect MetaNet Desktop Wallet" button
# Wallet popup should appear
# After authorization, should show "✅ Wallet connected successfully!"
```

### MetaNet Desktop Wallet Installation

**Required for frontend development:**

1. Download MetaNet Desktop Wallet from: https://fast.brc.dev/
2. Install and set up wallet
3. Get testnet BSV from faucet: https://faucet.bsvblockchain.org/
4. Your dApp will connect to this wallet via WalletClient

**Documentation:**
- WalletClient API: https://fast.brc.dev/
- BRC Specification: https://fast.brc.dev/
- Integration Guide: https://fast.brc.dev/

---

## Part 3: Testing Your Setup

### Backend Test

```bash
cd my-bsv-backend
npm run test
```

**Expected Output:**
```
=== BSV Backend Development Environment Test ===

✅ Private key generated
Address: 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
Network: testnet
✅ Transaction object created
✅ BSV SDK is working correctly!
🚀 Ready for backend development
```

### Frontend Test

```bash
cd my-bsv-dapp
npm start
```

**Expected Behavior:**
1. Browser opens at http://localhost:3000
2. Click "Connect MetaNet Desktop Wallet"
3. Wallet popup appears
4. Authorize connection
5. See "✅ Wallet connected successfully!"
6. Address displayed

---

## Additional Tools

### Block Explorers

**WhatsOnChain:**
- Mainnet: https://whatsonchain.com/
- Testnet: https://test.whatsonchain.com/

**Usage:**
- View transactions by TXID
- Check address balances
- Explore blocks and network stats
- Debug transaction issues

### Testnet Faucet

Get free testnet BSV for development:
- Official Faucet: https://faucet.bsvblockchain.org/
- Request 0.01 BSV to your testnet address
- Use for testing without real funds

### API Documentation

- **BSV SDK Docs**: https://bsv-blockchain.github.io/ts-sdk/
- **WalletClient Reference**: https://fast.brc.dev/
- **BRC Standards**: https://github.com/bitcoin-sv/BRCs

---

## Environment Variables Best Practices

### Backend (.env)

```env
# Network Configuration
NETWORK=testnet

# Database
DATABASE_URL=mongodb://localhost:27017/bsv-app

# API
PORT=3000
API_SECRET=your-secret-key-here

# Security (Development only - use HSM in production)
# MASTER_PRIVATE_KEY_WIF=cVrt...
ENCRYPTION_KEY=your-encryption-key-here
```

### Frontend (.env)

```env
# API Endpoint (if you have a backend)
REACT_APP_API_URL=http://localhost:3000

# Network
REACT_APP_NETWORK=testnet

# Note: Frontend has NO private keys
# All signing happens in user's wallet
```

**Security Rules:**
- ✅ Always add `.env` to `.gitignore`
- ✅ Use different `.env` files for dev/prod
- ✅ Never commit private keys
- ✅ Use environment variables for all secrets
- ✅ In backend production, use HSM or key vaults

---

## Troubleshooting

### "Cannot find module '@bsv/sdk'"

```bash
# Reinstall dependencies
rm -rf node_modules package-lock.json
npm install
```

### "WalletClient connection failed"

**Checklist:**
- ✅ MetaNet Desktop Wallet installed?
- ✅ Wallet unlocked?
- ✅ Testnet mode if using testnet?
- ✅ dApp origin authorized in wallet?

**Solution:**
```bash
# Check wallet is running
# Check browser console for detailed error
# Verify wallet extension is enabled
```

### TypeScript Errors

```bash
# Update TypeScript
npm install -D typescript@latest

# Regenerate tsconfig.json
npx tsc --init

# Check for type definition issues
npm install -D @types/node
```

### Permission Errors (Linux/macOS)

```bash
# Fix npm permissions
sudo chown -R $USER /usr/local/lib/node_modules
```

### Port Already in Use

```bash
# Backend (port 3000)
lsof -ti:3000 | xargs kill -9

# Frontend (port 3000 for React)
# Change port in package.json or:
PORT=3001 npm start
```

---

## Verification Checklist

### Backend Development Ready

- ✅ Node.js 18+ installed
- ✅ TypeScript 5.0+ installed
- ✅ BSV SDK installed
- ✅ Project structure created
- ✅ Test script runs successfully
- ✅ `.env` file configured
- ✅ Git initialized

### Frontend Development Ready

- ✅ Node.js 18+ installed
- ✅ React project created
- ✅ BSV SDK installed
- ✅ MetaNet Desktop Wallet installed
- ✅ Wallet connection test works
- ✅ Test component renders
- ✅ Git initialized

---

## Next Steps

Now that your environment is ready, continue to:

**For Both Paradigms:**
- [BSV Fundamentals](../bsv-fundamentals/) - Learn UTXO model, transactions, scripts

**Then Choose Your Path:**

**Backend Developers:**
- [Managing Wallets Server-Side](../first-wallet/) - Server-side wallet management
- [Building Transactions Programmatically](../first-transaction/) - Transaction creation

**Frontend Developers:**
- [WalletClient Integration](../wallet-client-integration/) - Connect to user wallets
- [Building dApps with WalletClient](../first-transaction/) - Request user signatures

---

## Related Resources

### Official Documentation
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk/)
- [WalletClient Reference](https://fast.brc.dev/)
- [Node.js Documentation](https://nodejs.org/docs)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)

### Tools
- [VS Code](https://code.visualstudio.com/)
- [WhatsOnChain Explorer](https://whatsonchain.com/)
- [MetaNet Desktop Wallet](https://fast.brc.dev/)
- [BSV Testnet Faucet](https://faucet.bsvblockchain.org/)

### Community
- [BSV Discord](https://discord.gg/bsv)
- [BSV Developers Reddit](https://www.reddit.com/r/bsvdevs/)
- [Stack Overflow (tag: bsv)](https://stackoverflow.com/questions/tagged/bsv)

---

**Your development environment is now ready! Let's learn BSV fundamentals.**
