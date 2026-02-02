# Deployment Guide

This guide covers how to set up, configure, and deploy the MessageBox Platform locally and in production.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Local Development Setup](#local-development-setup)
3. [Environment Configuration](#environment-configuration)
4. [Running the Application](#running-the-application)
5. [Production Deployment](#production-deployment)
6. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software

```bash
# Node.js (v18 or higher)
node --version  # Should be >= v18.0.0

# npm (comes with Node.js)
npm --version   # Should be >= 9.0.0

# MongoDB (v5 or higher)
mongod --version  # Should be >= v5.0.0

# Git (for cloning repository)
git --version
```

### BSV Wallet

You'll need a BSV wallet that supports the `@bsv/sdk` WalletClient interface:

- **Browser Extension Wallets**:
  - [Yours Wallet](https://github.com/yours-org/yours-wallet) (Recommended)
  - [HandCash Connect](https://handcash.io/)

- **Desktop Wallets**:
  - [Electrum SV](https://electrumsv.io/)

- **Mobile Wallets**:
  - [Simply Cash](https://simply.cash/)
  - [RelayX](https://relayx.com/)

**Note**: For testing, you can use testnet satoshis (free) from a BSV testnet faucet.

---

## Local Development Setup

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/messagebox-platform.git
cd messagebox-platform
```

### 2. Install Dependencies

#### Server Dependencies

```bash
cd server
npm install
```

**Key Dependencies**:
- `@bsv/sdk`: ^1.8.8
- `@bsv/message-box-client`: ^1.4.5
- `express`: ^4.18.2
- `mongodb`: ^6.3.0
- `cors`: ^2.8.5

#### Frontend Dependencies

```bash
cd ../frontend-next
npm install
```

**Key Dependencies**:
- `@bsv/sdk`: ^1.1.58
- `@bsv/message-box-client`: ^1.4.5
- `next`: ^15.1.6
- `react`: ^19.0.0
- `tailwindcss`: ^3

### 3. Set Up MongoDB

#### Option A: Local MongoDB

```bash
# Start MongoDB service
# macOS (using Homebrew)
brew services start mongodb-community

# Linux (systemd)
sudo systemctl start mongod

# Windows
# Use MongoDB Compass or start service from Services app
```

#### Option B: MongoDB Atlas (Cloud)

1. Create account at [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
2. Create a free cluster
3. Get connection string:
   ```
   mongodb+srv://username:password@cluster0.mongodb.net/messagebox_certifier
   ```

### 4. Configure Environment Variables

#### Server Environment (`.env`)

Create `server/.env`:

```env
# Server Configuration
PORT=3000
NODE_ENV=development

# MessageBox Network
MESSAGEBOX_HOST=https://messagebox.babbage.systems
BSV_NETWORK=mainnet

# MongoDB Connection
MONGO_URI=mongodb://localhost:27017
MONGO_DB=messagebox_certifier

# Session Configuration
SESSION_TIMEOUT_MINUTES=30

# Optional: Logging
LOG_LEVEL=info
```

#### Frontend Environment (`.env.local`)

Create `frontend-next/.env.local`:

```env
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:3000

# MessageBox Network
NEXT_PUBLIC_MESSAGEBOX_HOST=https://messagebox.babbage.systems

# Optional: Analytics
NEXT_PUBLIC_ANALYTICS_ID=
```

---

## Environment Configuration

### Environment Variables Explained

#### Server Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `PORT` | Server port number | 3000 | No |
| `NODE_ENV` | Environment (development/production) | development | No |
| `MESSAGEBOX_HOST` | MessageBox network URL | https://messagebox.babbage.systems | Yes |
| `BSV_NETWORK` | BSV network (mainnet/testnet) | mainnet | Yes |
| `MONGO_URI` | MongoDB connection string | mongodb://localhost:27017 | Yes |
| `MONGO_DB` | MongoDB database name | messagebox_certifier | Yes |
| `SESSION_TIMEOUT_MINUTES` | Session expiration time | 30 | No |

#### Frontend Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NEXT_PUBLIC_API_URL` | Backend API URL | http://localhost:3000 | Yes |
| `NEXT_PUBLIC_MESSAGEBOX_HOST` | MessageBox network URL | https://messagebox.babbage.systems | Yes |

**Important**: All frontend environment variables MUST be prefixed with `NEXT_PUBLIC_` to be accessible in the browser.

### Network Configuration

#### Mainnet vs Testnet

**Mainnet** (real BSV):
```env
BSV_NETWORK=mainnet
MESSAGEBOX_HOST=https://messagebox.babbage.systems
```

**Testnet** (free test BSV):
```env
BSV_NETWORK=testnet
MESSAGEBOX_HOST=https://testnet.messagebox.babbage.systems
```

**Get Testnet Coins**:
- [BSV Testnet Faucet](https://testnet.satoshisvision.network/)
- Request free testnet satoshis for development

---

## Running the Application

### Development Mode

#### 1. Start MongoDB

```bash
# Check if MongoDB is running
mongosh
# Should connect without errors

# If not running, start it:
brew services start mongodb-community  # macOS
sudo systemctl start mongod           # Linux
```

#### 2. Start Backend Server

```bash
cd server
npm run dev
```

**Expected Output**:
```
✓ MongoDB connected: messagebox_certifier
Server running on port 3000
Environment: development
MessageBox Host: https://messagebox.babbage.systems
```

**API Endpoints Available**:
- `GET http://localhost:3000/` - API info
- `GET http://localhost:3000/health` - Health check
- `POST http://localhost:3000/api/wallet/connect` - Connect wallet
- `POST http://localhost:3000/api/certify` - Certify identity
- `POST http://localhost:3000/api/initiate-payment` - Send payment
- `GET http://localhost:3000/api/certified-users` - List users

#### 3. Start Frontend

```bash
# In a new terminal
cd frontend-next
npm run dev
```

**Expected Output**:
```
▲ Next.js 15.1.6
- Local:        http://localhost:3001
- Environments: .env.local

✓ Ready in 2.3s
```

#### 4. Access Application

Open browser to: **http://localhost:3001**

### Production Mode

#### 1. Build Frontend

```bash
cd frontend-next
npm run build
```

**Expected Output**:
```
✓ Compiled successfully
✓ Linting and checking validity of types
✓ Collecting page data
✓ Generating static pages (4/4)
✓ Finalizing page optimization

Route (app)                              Size
┌ ○ /                                   150 kB
├ ○ /payments                           152 kB
├ ○ /receive                            151 kB
└ λ /api/[...path]                      0 B
```

#### 2. Start Production Servers

```bash
# Backend
cd server
npm start

# Frontend (in new terminal)
cd frontend-next
npm run start
```

---

## Production Deployment

### Option 1: Traditional Server (VPS/Dedicated)

#### 1. Server Requirements

- **OS**: Ubuntu 22.04 LTS or similar
- **RAM**: Minimum 2GB
- **Disk**: 20GB SSD
- **CPU**: 2 cores

#### 2. Setup Script

```bash
#!/bin/bash

# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install MongoDB
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | \
   sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/mongodb-6.gpg
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt update
sudo apt install -y mongodb-org

# Start MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod

# Install PM2 (process manager)
sudo npm install -g pm2

# Clone repository
git clone https://github.com/yourusername/messagebox-platform.git
cd messagebox-platform

# Setup backend
cd server
npm install
cp .env.example .env
# Edit .env with production values
nano .env

# Setup frontend
cd ../frontend-next
npm install
cp .env.local.example .env.local
# Edit .env.local with production values
nano .env.local

# Build frontend
npm run build

# Start services with PM2
cd ../server
pm2 start npm --name "messagebox-api" -- start

cd ../frontend-next
pm2 start npm --name "messagebox-web" -- start

# Save PM2 configuration
pm2 save
pm2 startup
```

#### 3. Nginx Reverse Proxy

```nginx
# /etc/nginx/sites-available/messagebox

server {
    listen 80;
    server_name yourdomain.com;

    # Frontend
    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Backend API
    location /api {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}
```

Enable site:
```bash
sudo ln -s /etc/nginx/sites-available/messagebox /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

#### 4. SSL Certificate (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

### Option 2: Docker Deployment

#### 1. Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # MongoDB
  mongodb:
    image: mongo:6.0
    container_name: messagebox-mongodb
    restart: always
    volumes:
      - mongodb_data:/data/db
    environment:
      MONGO_INITDB_DATABASE: messagebox_certifier
    ports:
      - "27017:27017"

  # Backend API
  backend:
    build:
      context: ./server
      dockerfile: Dockerfile
    container_name: messagebox-api
    restart: always
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MONGO_URI=mongodb://mongodb:27017
      - MONGO_DB=messagebox_certifier
      - MESSAGEBOX_HOST=https://messagebox.babbage.systems
      - BSV_NETWORK=mainnet
    depends_on:
      - mongodb

  # Frontend
  frontend:
    build:
      context: ./frontend-next
      dockerfile: Dockerfile
    container_name: messagebox-web
    restart: always
    ports:
      - "3001:3001"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:3000
      - NEXT_PUBLIC_MESSAGEBOX_HOST=https://messagebox.babbage.systems
    depends_on:
      - backend

volumes:
  mongodb_data:
```

#### 2. Backend Dockerfile

Create `server/Dockerfile`:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

#### 3. Frontend Dockerfile

Create `frontend-next/Dockerfile`:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build Next.js app
RUN npm run build

EXPOSE 3001

CMD ["npm", "start"]
```

#### 4. Deploy

```bash
# Build and start containers
docker-compose up -d

# View logs
docker-compose logs -f

# Stop containers
docker-compose down
```

### Option 3: Cloud Platforms

#### Vercel (Frontend Only)

```bash
cd frontend-next

# Install Vercel CLI
npm i -g vercel

# Deploy
vercel --prod
```

**Environment Variables** (set in Vercel dashboard):
- `NEXT_PUBLIC_API_URL` = your backend URL
- `NEXT_PUBLIC_MESSAGEBOX_HOST` = https://messagebox.babbage.systems

#### Heroku (Backend)

```bash
cd server

# Create Heroku app
heroku create messagebox-api

# Add MongoDB addon
heroku addons:create mongolab

# Set environment variables
heroku config:set MESSAGEBOX_HOST=https://messagebox.babbage.systems
heroku config:set BSV_NETWORK=mainnet

# Deploy
git push heroku main
```

#### Railway.app (Full Stack)

1. Connect GitHub repository
2. Create two services:
   - Backend: `server` directory
   - Frontend: `frontend-next` directory
3. Add MongoDB database
4. Set environment variables in dashboard

---

## Troubleshooting

### Common Issues

#### 1. Wallet Connection Fails

**Error**: `Failed to connect to wallet`

**Solutions**:
```bash
# Check if wallet extension is installed
# Open browser console and type:
window.bsv

# Should return object with wallet methods
# If undefined, install BSV wallet extension

# Check if site has permission
# Wallet settings → Permissions → Allow localhost
```

#### 2. MongoDB Connection Error

**Error**: `MongoServerError: connect ECONNREFUSED`

**Solutions**:
```bash
# Check if MongoDB is running
mongosh

# Start MongoDB if not running
brew services start mongodb-community  # macOS
sudo systemctl start mongod           # Linux

# Check connection string in .env
MONGO_URI=mongodb://localhost:27017
MONGO_DB=messagebox_certifier
```

#### 3. Session Expired Issues

**Error**: `Session expired. Please reconnect your wallet.`

**Cause**: Session timeout (default 30 minutes)

**Solutions**:
- Reconnect wallet (click "Connect Wallet" button)
- Increase timeout in `server/.env`:
  ```env
  SESSION_TIMEOUT_MINUTES=60
  ```

#### 4. Payment Fails

**Error**: `Failed to send payment`

**Possible Causes**:
1. Insufficient balance
2. Recipient not certified
3. Network issues

**Solutions**:
```bash
# Check wallet balance
# Open wallet app and verify satoshi balance

# Verify recipient is certified
curl http://localhost:3000/api/certified-users

# Check logs
cd server
npm run dev
# Look for error details in console
```

#### 5. Internalization Error

**Error**: `The paymentRemittance parameter must be valid for protocol wallet payment`

**Cause**: Trying to internalize non-BRC-29 outputs (like change)

**Solution**: Only internalize recipient payment outputs, not sender's change:
```typescript
// ✅ CORRECT: Recipient internalizes payment
await walletClient.internalizeAction({
  tx: token.transaction,
  outputs: [{
    outputIndex: 0,  // Payment output only
    protocol: 'wallet payment',
    paymentRemittance: {
      senderIdentityKey,
      derivationPrefix,
      derivationSuffix
    }
  }]
})

// ❌ WRONG: Don't internalize change outputs
// Change is auto-tracked by sender's wallet
```

#### 6. Next.js Build Fails

**Error**: `Module not found: Can't resolve '@bsv/sdk'`

**Solution**:
```bash
# Clear cache and reinstall
rm -rf node_modules .next
npm install
npm run build
```

#### 7. CORS Errors

**Error**: `Access to fetch blocked by CORS policy`

**Solution** (server `index.ts`):
```typescript
import cors from 'cors'

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3001',
  credentials: true
}))
```

### Debug Mode

Enable detailed logging:

#### Backend
```env
LOG_LEVEL=debug
NODE_ENV=development
```

#### Frontend
```typescript
// Enable in walletClient.ts, paymentClient.ts, etc.
const client = new PeerPayClient({
  walletClient,
  messageBoxHost,
  enableLogging: true  // ← Enable this
})
```

### Performance Monitoring

```bash
# Backend metrics
pm2 monit

# Database stats
mongosh
use messagebox_certifier
db.stats()

# Frontend analytics
# Check browser DevTools → Network → Performance
```

---

## Security Considerations

### Production Checklist

- [ ] Use HTTPS (SSL certificate installed)
- [ ] Set strong MongoDB password
- [ ] Use environment variables (never hardcode secrets)
- [ ] Enable rate limiting on API endpoints
- [ ] Implement session validation
- [ ] Set secure CORS policy
- [ ] Regular dependency updates (`npm audit`)
- [ ] Enable MongoDB authentication
- [ ] Use reverse proxy (Nginx/Apache)
- [ ] Implement logging and monitoring
- [ ] Regular backups of MongoDB

### Rate Limiting Example

```typescript
// server/src/middleware/rateLimit.ts
import rateLimit from 'express-rate-limit'

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: 'Too many requests, please try again later'
})

// Apply to routes
app.use('/api/', apiLimiter)
```

---

## Monitoring & Maintenance

### Health Checks

```bash
# Backend health
curl http://localhost:3000/health

# MongoDB health
mongosh --eval "db.adminCommand('ping')"

# Frontend health
curl http://localhost:3001
```

### Log Rotation

```bash
# Using PM2
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
```

### Backup Strategy

```bash
# Backup MongoDB
mongodump --db messagebox_certifier --out /backup/$(date +%Y%m%d)

# Automated daily backup (cron)
0 2 * * * mongodump --db messagebox_certifier --out /backup/$(date +\%Y\%m\%d)
```

---

## Next Steps

📖 Return to [BSV Patterns Guide](./BSV_PATTERNS_GUIDE.md)

📖 See [Code Examples](./CODE_EXAMPLES.md) for implementation details

🚀 **Production Ready**: Your MessageBox Platform is now deployed!

For support, please open an issue on GitHub or contact the development team.
