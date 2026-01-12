# Project: Token Frontend UI

**React Application with BSV Token Management**

Build a token management interface where users can mint fungible tokens, transfer them to other users, and view their token balances. This project demonstrates BSV patterns including PushDrop tokens (BRC-48), key derivation (BRC-42), BEEF transactions (BRC-62), and overlay service integration.

---

## What You'll Build

A production-ready token management interface featuring:

- Token minting with custom metadata
- Peer-to-peer token transfers via MessageBox
- Live token wallet with balance tracking
- Overlay service validation before broadcast
- BSV Desktop Wallet integration

---

## Learning Objectives

By completing this project, you will learn:

- **PushDrop Tokens** - Creating fungible tokens with BRC-48
- **BRC-42 Key Derivation** - Privacy-enhanced address generation
- **WalletClient** - Frontend wallet integration with `createAction` and `signAction`
- **BEEF Format** - Transaction packaging with SPV proofs (BRC-62)
- **Two-Phase Signing** - Wallet BSV inputs + manual token inputs
- **Overlay Services** - Protocol validation before blockchain broadcast
- **MessageBox** - Off-chain peer-to-peer messaging

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│      BSV Desktop Wallet (User)          │
│  - Private key management               │
│  - Transaction signing                  │
│  - UTXO tracking                        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│       Frontend (React + Vite)           │
│  - WalletClient integration             │
│  - Token minting (PushDrop)             │
│  - Token transfers (two-phase signing)  │
│  - MessageBox for P2P transfers         │
│  - Token wallet viewer                  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         Overlay Service                 │
│  - Validate token protocol rules        │
│  - Accept/reject outputs                │
│  - Index admitted tokens                │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           BSV Blockchain                │
│  - Token mint transactions              │
│  - Token transfer transactions          │
└─────────────────────────────────────────┘
```

---

## Key Patterns

### 1. Wallet Integration

Connect to BSV Desktop Wallet using React context:

```typescript
import { WalletClient } from '@bsv/sdk'

const walletClient = new WalletClient()

// Get user's identity key
const { publicKey } = await walletClient.getPublicKey({
  identityKey: true
})
```

### 2. Token Minting with PushDrop

Create fungible tokens using the PushDrop protocol:

```typescript
import { PushDrop, Utils, Random } from '@bsv/sdk'

// Define 3-field token structure
const fields = [
  Utils.toArray('___mint___', 'utf8'),           // Field 0: Mint marker
  Utils.Writer().writeUInt64LEBn(amount),        // Field 1: Amount (64-bit)
  Utils.toArray(JSON.stringify(metadata), 'utf8') // Field 2: Metadata
]

const token = new PushDrop(wallet)
const protocolID = [2, 'tokendemo']
const keyID = Utils.toBase64(Random(8))  // Random 8-byte ID

// Create privacy-enhanced locking script (BRC-42)
const lockingScript = await token.lock(
  fields,
  protocolID,
  keyID,
  'self',      // Counterparty
  true,        // forSelf
  false
)

// Create transaction
const result = await wallet.createAction({
  outputs: [{
    satoshis: 1,
    lockingScript: lockingScript.toHex(),
    basket: 'demotokens3',
    customInstructions: JSON.stringify({ protocolID, keyID, counterparty: 'self' }),
    tags: ['demotokens3', 'mint', tokenLabel]
  }]
})
```

**Key Concepts**:
- **Field 0**: Token ID (`'___mint___'` at mint, then becomes `txid.vout`)
- **Field 1**: Amount as 64-bit little-endian integer
- **Field 2**: Custom metadata (JSON)
- **customInstructions**: Store derivation params for future unlocking
- **BRC-42**: Each output uses a unique derived address for privacy

> **Reference**: [PushDrop](https://docs.bsvblockchain.org/) | [BRC-42](https://brc.dev/42)

### 3. Two-Phase Token Signing

Token transfers require manual signing because wallet can't derive PushDrop keys:

**Phase 1: Create Transaction with Inputs**

```typescript
// List token UTXOs
const outputs = await wallet.listOutputs({
  basket: 'demotokens3',
  include: 'entire transactions',
  includeCustomInstructions: true
})

// Decode token amounts
const decoded = PushDrop.decode(LockingScript.fromHex(output.lockingScript))
const amount = new Utils.Reader(decoded.fields[1]).readUInt64LEBn()

// Build BEEF with parent transactions (SPV proofs)
const beef = Beef.fromBinary(Utils.toArray(output.beef, 'base64'))

// Create transaction (wallet only estimates unlocking script length)
const response = await wallet.createAction({
  inputBEEF: beef.toBinary(),
  inputs: [{
    outpoint: `${output.txid}.${output.vout}`,
    unlockingScriptLength: 73  // Estimate for PushDrop
  }],
  outputs: [
    recipientOutput,
    changeOutput  // Send remaining balance back to self
  ]
})
```

**Phase 2: Sign Token Inputs**

```typescript
// Parse signable transaction
const txToSign = Transaction.fromBEEF(response.signableTransaction.tx)

// Sign each token input using stored customInstructions
for (let i = 0; i < numberOfInputsUsed; i++) {
  const derivationParams = JSON.parse(customInstructions[i])

  txToSign.inputs[i].unlockingScriptTemplate = new PushDrop(wallet).unlock(
    derivationParams.protocolID,
    derivationParams.keyID,
    derivationParams.counterparty
  )
}

await txToSign.sign()
```

**Phase 3: Submit Signatures**

```typescript
await wallet.signAction({
  reference: response.signableTransaction.reference,
  spends: response.signableTransaction.spends
})
```

> **Reference**: [Transaction Signing](https://docs.bsvblockchain.org/)

### 4. Overlay Validation

Validate with overlay service BEFORE final broadcast:

```typescript
import { HTTPSOverlayBroadcastFacilitator } from '@bsv/sdk'

const overlay = new HTTPSOverlayBroadcastFacilitator(undefined, true)

const taggedBEEF = {
  beef: tx.toBEEF(),
  topics: ['tm_tokendemo']  // Topic manager ID
}

const overlayResponse = await overlay.send(
  'http://localhost:3000/submit',
  taggedBEEF
)

// Check if overlay accepted outputs
if (overlayResponse['tm_tokendemo'].outputsToAdmit.length === 0) {
  throw new Error('Overlay rejected transaction')
}
```

**Why Overlay First?**
- Validates protocol-specific rules (balance conservation, no duplicate mints)
- Prevents invalid transactions from reaching blockchain
- Indexes valid tokens for fast lookup

> **Reference**: [Overlay Services](  overlays/0022)

### 5. MessageBox Token Transfer

Send token notification off-chain for recipient to claim:

```typescript
import { MessageBoxClient } from '@bsv/sdk'

const mbClient = new MessageBoxClient({
  host: 'https://messagebox.babbage.systems',
  walletClient: wallet
})

// Send token details to recipient
await mbClient.sendMessage({
  recipient: recipientIdentityKey,
  messageBox: 'demotokenpayments',
  body: {
    tokenId,
    amount,
    transaction: tx,  // Full BEEF with SPV proofs
    keyID,
    protocolID,
    sender: myIdentityKey
  }
})
```

**Recipient Claims Token**:

```typescript
// Recipient retrieves messages
const messages = await mbClient.getMessages({
  messageBox: 'demotokenpayments'
})

// Import token transaction
await wallet.internalizeAction({
  tx: message.body.transaction,
  outputs: [{
    outputIndex: recipientOutputIndex,
    protocol: 'basket insertion',
    insertionRemittance: { basket: 'demotokens3' },
    customInstructions: JSON.stringify({
      protocolID: message.body.protocolID,
      keyID: message.body.keyID,
      counterparty: message.body.sender
    })
  }]
})
```

> **Reference**: [MessageBox](https://hub.bsvblockchain.org/brc/peer-to-peer/0033)

### 6. Token Wallet Viewing

Display user's token balances by aggregating UTXOs:

```typescript
const outputs = await wallet.listOutputs({
  basket: 'demotokens3',
  include: 'locking scripts',
  includeCustomInstructions: true
})

// Aggregate by tokenId
const balances = new Map()

for (const output of outputs.outputs) {
  const decoded = PushDrop.decode(LockingScript.fromHex(output.lockingScript))

  let tokenId = Utils.toUTF8(decoded.fields[0])
  if (tokenId === '___mint___') {
    tokenId = `${output.txid}.${output.vout}`  // First mint creates tokenId
  }

  const amount = new Utils.Reader(decoded.fields[1]).readUInt64LEBn()
  const metadata = JSON.parse(Utils.toUTF8(decoded.fields[2]))

  balances.set(tokenId, {
    amount: (balances.get(tokenId)?.amount || 0) + amount,
    metadata
  })
}
```

---

## Important Concepts

### Token Structure

3-field format embedded in PushDrop locking scripts:

```typescript
{
  fields: [
    tokenId,    // UTF-8 string: '___mint___' or 'txid.vout'
    amount,     // Uint64LE (8 bytes)
    metadata    // JSON: [{ trait: "name", value: "MyToken" }]
  ]
}
```

### Token Lifecycle

1. **Mint**: Create with `'___mint___'` marker → tokenId becomes `txid.vout`
2. **Transfer**: Spend UTXO → create new UTXO with same tokenId
3. **Split**: Create multiple outputs (e.g., recipient + change)
4. **Merge**: Spend multiple UTXOs of same tokenId → create single output

### Privacy Model

Every token output uses a **unique derived address** via BRC-42:

- **Protocol ID**: `[2, 'tokendemo']`
- **Key ID**: Random 8 bytes per output
- **Counterparty**: Recipient's identity key or `'self'`

This prevents:
- Address reuse
- Balance tracking by external observers
- Linking multiple token UTXOs to same owner

### BEEF Format (BRC-62)

Transactions are packaged with **SPV proofs** for validation:

```typescript
{
  transaction: rawTx,
  parentTransactions: [parentTx1, parentTx2],
  merklePaths: [proof1, proof2]
}
```

Wallet validates inputs without needing full blockchain.

### Basket Organization

Use separate basket to isolate token UTXOs from BSV:

```typescript
basket: 'demotokens3'  // Only token outputs
```

Prevents accidentally spending tokens as BSV.

### Critical: customInstructions

MUST store derivation parameters to recreate unlocking scripts:

```typescript
customInstructions: JSON.stringify({
  protocolID: [2, 'tokendemo'],
  keyID: 'base64RandomBytes',
  counterparty: 'self' | 'recipientIdentityKey'
})
```

Without this, tokens become unspendable.

---

## Project Structure

```
frontend/
├── src/
│   ├── App.tsx                    # Main application component
│   ├── context/
│   │   └── WalletContext.tsx      # Wallet provider (WalletClient setup)
│   ├── components/
│   │   ├── CreateTokens.tsx       # Token minting UI
│   │   ├── SendTokens.tsx         # Token transfer UI (two-phase signing)
│   │   ├── ReceiveTokens.tsx      # Token claiming UI (MessageBox)
│   │   ├── TokenWallet.tsx        # Balance viewer (listOutputs)
│   │   └── TokenDemo.tsx          # Main orchestrator component
│   └── lib/
│       └── utils.ts               # Helper functions
├── vite.config.ts                 # Vite configuration
└── package.json                   # Dependencies
```

---

## Setup

1. **Install BSV Desktop Wallet**

   Download from [https://desktop.bsvb.tech/](https://desktop.bsvb.tech/)

2. **Install dependencies**
   ```bash
   cd frontend
   npm install
   ```

3. **Configure overlay URL**

   Create `.env` file:
   ```
   VITE_OVERLAY_URL=http://localhost:3000/submit
   ```

4. **Start development server**
   ```bash
   npm run dev
   ```

5. **Open browser**

   Navigate to [http://localhost:5173](http://localhost:5173)

---

## Summary

This project demonstrates:

- **PushDrop Tokens (BRC-48)** - Fungible token creation with 3-field structure
- **BRC-42 Key Derivation** - Privacy-enhanced addresses for each output
- **Two-Phase Signing** - Wallet BSV signing + manual token input signing
- **BEEF Format (BRC-62)** - Transaction packaging with SPV proofs
- **Overlay Integration** - Protocol validation before blockchain broadcast
- **MessageBox** - Off-chain peer-to-peer token transfer notifications
- **Basket Management** - Organizing token UTXOs separately from BSV
- **customInstructions** - Storing derivation parameters for future spending

These patterns form the foundation for building privacy-preserving fungible token systems on BSV.

---

## Related Resources

- [BRC-42: Key Derivation Protocol](https://hub.bsvblockchain.org/brc/key-derivation/0042)
- [BRC-48: PushDrop Token Standard](https://hub.bsvblockchain.org/brc/scripts/0048)
- [BRC-62: BEEF Transaction Format](https://hub.bsvblockchain.org/brc/transactions/0062)
- [BSV SDK Documentation](https://docs.bsvblockchain.org/)
- [Overlay Services (BRC-57)](https://hub.bsvblockchain.org/brc/overlays/0088)
