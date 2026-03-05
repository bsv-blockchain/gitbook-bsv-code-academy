# @bsv/simple

A high-level TypeScript library that makes BSV blockchain development simple. Build wallets, send payments, create tokens, issue credentials, and more — in just a few lines of code.

## What is @bsv/simple?

`@bsv/simple` wraps the low-level `@bsv/sdk` into a clean, modular API. Instead of manually constructing locking scripts, managing key derivation, and handling transaction internalization, you call methods like `wallet.pay()`, `wallet.createToken()`, and `wallet.inscribeText()`.

## What can you build?

| Feature | Description |
|---------|-------------|
| **Payments** | Send BSV to any identity key via BRC-29 peer-to-peer payments |
| **Multi-Output Transactions** | Combine P2PKH payments, OP_RETURN data, and PushDrop tokens in a single transaction |
| **Encrypted Tokens** | Create, transfer, and redeem PushDrop tokens with encrypted payloads |
| **Inscriptions** | Write text, JSON, or file hashes permanently to the blockchain |
| **MessageBox P2P** | Send and receive payments and tokens peer-to-peer via MessageBox |
| **Certification** | Issue and manage BSV certificates with a standalone Certifier |
| **Verifiable Credentials** | W3C-compatible VCs backed by BSV certificates, with on-chain revocation |
| **DIDs** | Generate and resolve `did:bsv:` Decentralized Identifiers |
| **Overlay Networks** | Broadcast to and query SHIP/SLAP overlay services |
| **Server Wallet** | Run a backend wallet for automated operations and funding flows |

## Browser vs Server

The library has two entry points:

- **`@bsv/simple`** (Default) Browser-safe. Uses `WalletClient` from `@bsv/sdk` to connect to the user's wallet on the client side. Will not pull in any server-only dependencies.
- **`@bsv/simple/server`** — Uses `@bsv/wallet-toolbox` to run a server-side wallet from a private key. Used for agents, or servers receiving payments.

Both entry points provide the same API surface — the only difference is how they connect to the underlying wallet.

## A taste of the API

```typescript
import { createWallet } from '@bsv/simple/browser'

// Connect to the user's wallet
const wallet = await createWallet()

// Send a payment
await wallet.pay({ to: recipientKey, satoshis: 1000, memo: 'Coffee' })

// Create an encrypted token
await wallet.createToken({ data: { type: 'loyalty', points: 50 }, basket: 'rewards' })

// Inscribe text on-chain
await wallet.inscribeText('Hello BSV!')

// Get your DID
const did = wallet.getDID()
// { id: 'did:bsv:02abc...', ... }
```

## MCP Server for AI Assistants

`@bsv/simple` ships with a companion **Model Context Protocol (MCP) server** that gives AI coding assistants (Claude Code, Cursor, Copilot, etc.) structured knowledge about the library and the ability to generate integration code.

### Quick Start — Claude Code (recommended)

```bash
claude mcp add simple-mcp -- npx -y @bsv/simple-mcp
```

That's it — Claude will automatically download and run the MCP server via npx.

### Docker (alternative)

```bash
docker build -t simple-mcp .
```

Add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "simple": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "simple-mcp"]
    }
  }
}
```

### What the MCP Server Provides

**Resources** — read-only knowledge the AI can consult:

| URI | Description |
|-----|-------------|
| `simple://api/wallet` | WalletCore + BrowserWallet + ServerWallet methods |
| `simple://api/tokens` | Token create/list/send/redeem/messagebox |
| `simple://api/inscriptions` | Text/JSON/hash inscriptions |
| `simple://api/messagebox` | Certification, payments, identity registry |
| `simple://api/certification` | Certifier, certificates, revocation |
| `simple://api/did` | DID class, wallet DID methods |
| `simple://api/credentials` | Schema, Issuer, VC/VP, revocation stores |
| `simple://api/overlay` | Overlay, SHIP/SLAP, broadcasting |
| `simple://guide/nextjs` | Complete Next.js integration guide |

**Tools** — functions the AI can call to generate code:

| Tool | Description |
|------|-------------|
| `scaffold_nextjs_config` | Generate next.config.ts + package.json for BSV apps |
| `generate_wallet_setup` | Wallet initialization code (browser or server) |
| `generate_payment_handler` | Payment handler functions |
| `generate_token_handler` | Token CRUD operations |
| `generate_inscription_handler` | OP_RETURN inscription handlers |
| `generate_messagebox_setup` | MessageBox P2P integration |
| `generate_server_route` | Next.js API route for server wallet |
| `generate_credential_issuer` | CredentialIssuer setup with schema |
| `generate_did_integration` | DID integration code |

**Prompts** — pre-built conversation templates:

| Prompt | Description |
|--------|-------------|
| `integrate_simple` | Full integration walkthrough |
| `add_bsv_feature` | Feature-specific code generation |
| `debug_simple` | Debugging help for common issues |

## Next Steps

- [Quick Start](quick-start.md) — Get running in 5 minutes
- [Installation](installation.md) — Detailed setup instructions
- [Architecture](architecture.md) — How the library is built
