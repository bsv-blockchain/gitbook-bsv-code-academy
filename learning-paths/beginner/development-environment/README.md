# Development Environment Setup

## Overview

In this module, you'll set up a complete development environment for BSV blockchain development using the TypeScript SDK.

## Prerequisites

- Computer running Windows, macOS, or Linux
- Basic command line knowledge
- Text editor or IDE

## Installation Steps

### 1. Install Node.js

The BSV TypeScript SDK requires Node.js version 16 or higher.

#### macOS
```bash
# Using Homebrew
brew install node

# Verify installation
node --version
npm --version
```

#### Windows
1. Download installer from [nodejs.org](https://nodejs.org/)
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

### 2. Install TypeScript

```bash
# Install TypeScript globally
npm install -g typescript

# Verify installation
tsc --version
```

### 3. Set Up Your IDE

We recommend **Visual Studio Code** for TypeScript development:

1. Download from [code.visualstudio.com](https://code.visualstudio.com/)
2. Install these recommended extensions:
   - **ESLint**: Code quality
   - **Prettier**: Code formatting
   - **TypeScript Import Sorter**: Organize imports
   - **GitLens**: Git integration

#### VS Code Settings
Create `.vscode/settings.json` in your project:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "typescript.preferences.importModuleSpecifier": "relative"
}
```

### 4. Create Your First BSV Project

```bash
# Create project directory
mkdir my-bsv-app
cd my-bsv-app

# Initialize npm project
npm init -y

# Install BSV SDK
npm install @bsv/sdk

# Install development dependencies
npm install -D typescript @types/node ts-node

# Create TypeScript configuration
npx tsc --init
```

### 5. Configure TypeScript

Edit `tsconfig.json`:
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
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### 6. Create Project Structure

```bash
# Create directories
mkdir src
mkdir src/utils
mkdir src/examples

# Create main file
touch src/index.ts
```

### 7. Write Your First BSV Code

Create `src/index.ts`:
```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate a private key
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()
const address = publicKey.toAddress()

console.log('=== BSV Development Environment Test ===')
console.log('Private Key (WIF):', privateKey.toWif())
console.log('Public Key:', publicKey.toHex())
console.log('Address:', address)
console.log('\n✅ BSV SDK is working correctly!')
```

### 8. Add NPM Scripts

Edit `package.json`:
```json
{
  "name": "my-bsv-app",
  "version": "1.0.0",
  "scripts": {
    "start": "ts-node src/index.ts",
    "build": "tsc",
    "dev": "ts-node --watch src/index.ts"
  },
  "dependencies": {
    "@bsv/sdk": "^1.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "ts-node": "^10.9.0",
    "typescript": "^5.0.0"
  }
}
```

### 9. Test Your Setup

```bash
# Run the test script
npm start

# You should see output with a private key, public key, and address
```

## Additional Tools

### WhatsOnChain (Block Explorer)
Browse the BSV blockchain:
- Mainnet: https://whatsonchain.com/
- Testnet: https://test.whatsonchain.com/

### BSV Testnet Faucet
Get free testnet BSV for development:
- https://faucet.bsvblockchain.org/

### Git Version Control
```bash
# Initialize git repository
git init

# Create .gitignore
cat > .gitignore << EOF
node_modules/
dist/
.env
*.log
.DS_Store
EOF

# First commit
git add .
git commit -m "Initial BSV project setup"
```

## Environment Variables

For managing sensitive data (like private keys), use environment variables:

1. Install dotenv:
```bash
npm install dotenv
npm install -D @types/dotenv
```

2. Create `.env` file:
```
# Never commit this file!
PRIVATE_KEY_WIF=your-private-key-here
NETWORK=testnet
```

3. Add to `.gitignore`:
```
.env
```

4. Use in code:
```typescript
import * as dotenv from 'dotenv'
dotenv.config()

const privateKeyWif = process.env.PRIVATE_KEY_WIF
```

## Project Structure Best Practices

```
my-bsv-app/
├── src/
│   ├── index.ts          # Main entry point
│   ├── config/           # Configuration files
│   ├── services/         # Business logic
│   ├── utils/            # Helper functions
│   ├── models/           # Data models
│   └── examples/         # Example code
├── dist/                 # Compiled JavaScript (gitignored)
├── node_modules/         # Dependencies (gitignored)
├── .env                  # Environment variables (gitignored)
├── .gitignore
├── package.json
├── tsconfig.json
└── README.md
```

## Troubleshooting

### "Cannot find module '@bsv/sdk'"
```bash
# Reinstall dependencies
rm -rf node_modules package-lock.json
npm install
```

### TypeScript Errors
```bash
# Update TypeScript
npm install -D typescript@latest

# Regenerate tsconfig.json
npx tsc --init
```

### Permission Errors (Linux/macOS)
```bash
# Fix npm permissions
sudo chown -R $USER /usr/local/lib/node_modules
```

## Next Steps

Now that your environment is ready, let's learn about BSV fundamentals!

Continue to: [BSV Fundamentals](../bsv-fundamentals/README.md)

## Related Resources

- [Node.js Documentation](https://nodejs.org/docs)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [BSV SDK Documentation](https://hub.bsvblockchain.org/bsv-skills-center/guides/sdks/ts)

## Checklist

Before moving forward, ensure you have:
- ✅ Node.js 16+ installed
- ✅ TypeScript installed
- ✅ IDE configured
- ✅ BSV SDK project created
- ✅ Successfully run your first BSV code
- ✅ Git initialized (recommended)

Ready to learn BSV fundamentals? Let's go!
