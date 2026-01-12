# Whitelabel Token Frontend

Build a token management interface using BSV SDK's PushDrop, WalletClient, and MessageBox.

---

## What You'll Build

- Token minting with PushDrop (BRC-48)
- P2P transfers via MessageBox (BRC-33)
- Balance tracking with UTXO aggregation
- Two-phase transaction signing

**Token Structure (3 fields):**
```typescript
[
  tokenId,    // UTF-8: '___mint___' or 'txid.vout'
  amount,     // Uint64LE (8 bytes)
  metadata    // JSON string
]
```

---

## 📦 Complete Code Example

The full working implementation of this token frontend is available on GitHub:

**Repository**: [github.com/sirdeggen/demo-day/tree/main/tokenization](https://github.com/sirdeggen/demo-day/tree/main/tokenization)

- **Frontend Code**: `/frontend` folder
- **Overlay Service Code**: `/overlay` folder

You can clone the repository and explore the complete implementation alongside this tutorial.

---

## Setup

```bash
npm create vite@latest my-token-app -- --template react-ts
cd my-token-app
npm install @bsv/sdk sonner
```

Create `.env`:
```env
VITE_MESSAGEBOX_URL=https://messagebox.babbage.systems
```

---

## 1. Wallet Context

`src/context/WalletContext.tsx`:

```typescript
import { createContext, useContext, useState, useEffect } from 'react'
import { WalletClient, MessageBoxClient } from '@bsv/sdk'

const WalletContext = createContext<{
  wallet: WalletClient | null
  identityKey: string | null
  messageBoxClient: MessageBoxClient | null
  isInitialized: boolean
}>({ wallet: null, identityKey: null, messageBoxClient: null, isInitialized: false })

export const WalletProvider = ({ children }: { children: React.ReactNode }) => {
  const [wallet, setWallet] = useState<WalletClient | null>(null)
  const [identityKey, setIdentityKey] = useState<string | null>(null)
  const [messageBoxClient, setMessageBoxClient] = useState<MessageBoxClient | null>(null)
  const [isInitialized, setIsInitialized] = useState(false)

  useEffect(() => {
    const init = async () => {
      const w = new WalletClient()
      const { publicKey } = await w.getPublicKey({ identityKey: true })
      const mb = new MessageBoxClient({ host: import.meta.env.VITE_MESSAGEBOX_URL, walletClient: w })
      setWallet(w)
      setIdentityKey(publicKey)
      setMessageBoxClient(mb)
      setIsInitialized(true)
    }
    init()
  }, [])

  return <WalletContext.Provider value={{ wallet, identityKey, messageBoxClient, isInitialized }}>{children}</WalletContext.Provider>
}

export const useWallet = () => useContext(WalletContext)
```

> **BSV SDK Pattern**: WalletClient connects to BSV Desktop Wallet automatically. No API keys needed.

---

## 2. Token Minting

`src/components/CreateTokens.tsx`:

```typescript
import { WalletClient, PushDrop, Utils, Random, BigNumber } from '@bsv/sdk'
import { toast } from 'sonner'

export function CreateTokens({ wallet }: { wallet: WalletClient }) {
  const handleMint = async (label: string, amount: string) => {
    const token = new PushDrop(wallet)
    const protocolID: [0 | 1 | 2, string] = [2, 'mytoken']
    const keyID = Utils.toBase64(Random(8))

    // Build 3-field structure
    const fields = [
      Utils.toArray('___mint___', 'utf8'),                              // Field 0: Mint marker
      new Utils.Writer().writeUInt64LEBn(new BigNumber(amount)).toArray(), // Field 1: Amount (Uint64LE)
      Utils.toArray(JSON.stringify([{ name: 'label', value: label }]), 'utf8') // Field 2: Metadata
    ]

    // Create privacy-enhanced locking script (BRC-42)
    const lockingScript = await token.lock(fields, protocolID, keyID, 'self', true, false)

    const response = await wallet.createAction({
      description: `Mint ${amount} ${label}`,
      outputs: [{
        satoshis: 1,
        lockingScript: lockingScript.toHex(),
        basket: 'mytokens',
        customInstructions: JSON.stringify({ protocolID, keyID, counterparty: 'self' }),
        tags: ['mytokens', 'mint', label]
      }],
      options: { randomizeOutputs: false }
    })

    toast.success(`Minted ${amount} ${label} tokens!`)
  }

  // ... UI code ...
}
```

**Critical Points:**
- `'___mint___'` marker creates new token → tokenId becomes `txid.vout`
- `customInstructions` stores derivation params (required for spending)
- `randomizeOutputs: false` ensures predictable output indices

> **Reference**: [BRC-48: PushDrop](https://hub.bsvblockchain.org/brc/scripts/0048), [BRC-42: Key Derivation](https://hub.bsvblockchain.org/brc/key-derivation/0042)

---

## 3. Balance Viewer

`src/components/TokenWallet.tsx`:

```typescript
import { WalletClient, PushDrop, LockingScript, Utils } from '@bsv/sdk'

export function TokenWallet({ wallet }: { wallet: WalletClient }) {
  const loadBalances = async () => {
    const outputs = await wallet.listOutputs({
      basket: 'mytokens',
      include: 'locking scripts',
      limit: 1000
    })

    const balanceMap = new Map<string, { amount: number, label: string }>()

    for (const output of outputs.outputs) {
      const script = LockingScript.fromHex(output.lockingScript!)
      const decoded = PushDrop.decode(script)

      // Extract tokenId
      let tokenId = Utils.toUTF8(decoded.fields[0])
      if (tokenId === '___mint___') tokenId = output.outpoint

      // Extract amount
      const amount = Number(new Utils.Reader(decoded.fields[1]).readUInt64LEBn())

      // Extract metadata
      const metadata = JSON.parse(Utils.toUTF8(decoded.fields[2]))
      const label = metadata[0]?.value || 'Unknown'

      // Aggregate balance
      const current = balanceMap.get(tokenId) || { amount: 0, label }
      balanceMap.set(tokenId, { amount: current.amount + amount, label })
    }

    return balanceMap
  }

  // ... UI code ...
}
```

**Pattern**: Aggregate all UTXOs by tokenId to get total balance per token type.

---

## 4. Token Transfer (Two-Phase Signing)

`src/components/SendTokens.tsx`:

```typescript
import { WalletClient, PushDrop, Transaction, Beef, Utils, Random, BigNumber } from '@bsv/sdk'
import { MessageBoxClient } from '@bsv/sdk'

export function SendTokens({ wallet, messageBoxClient }: { wallet: WalletClient, messageBoxClient: MessageBoxClient }) {
  const handleSend = async (tokenId: string, sendAmount: number, recipient: string, label: string) => {
    const token = new PushDrop(wallet)
    const protocolID: [0 | 1 | 2, string] = [2, 'mytoken']
    const recipientKeyID = Utils.toBase64(Random(8))
    const changeKeyID = Utils.toBase64(Random(8))

    // 1. Collect UTXOs
    const outputs = await wallet.listOutputs({
      basket: 'mytokens',
      include: 'entire transactions',
      includeCustomInstructions: true
    })

    const beef = new Beef()
    let inAmountTotal = 0
    const inputs = []
    const customInstructions: string[] = []

    for (const output of outputs.outputs) {
      if (inAmountTotal >= sendAmount) break

      const [txid, vout] = output.outpoint.split('.')
      const tx = Transaction.fromBEEF(Utils.toArray(output.beef, 'base64'))
      const script = tx.outputs[Number(vout)].lockingScript
      const decoded = PushDrop.decode(script)

      let outTokenId = Utils.toUTF8(decoded.fields[0])
      if (outTokenId === '___mint___') outTokenId = output.outpoint
      if (outTokenId !== tokenId) continue

      const amount = Number(new Utils.Reader(decoded.fields[1]).readUInt64LEBn())
      inAmountTotal += amount

      beef.mergeBeef(tx.toBEEF())
      inputs.push({ outpoint: output.outpoint, unlockingScriptLength: 73 })
      customInstructions.push(output.customInstructions!)
    }

    // 2. Create recipient output
    const recipientFields = [
      Utils.toArray(tokenId, 'utf8'),
      new Utils.Writer().writeUInt64LEBn(new BigNumber(sendAmount)).toArray(),
      Utils.toArray(JSON.stringify([{ name: 'label', value: label }]), 'utf8')
    ]
    const recipientScript = await token.lock(recipientFields, protocolID, recipientKeyID, recipient, false, false)

    const txOutputs = [{ satoshis: 1, lockingScript: recipientScript.toHex() }]

    // 3. Create change output (if needed)
    const changeAmount = inAmountTotal - sendAmount
    if (changeAmount > 0) {
      const changeFields = [
        Utils.toArray(tokenId, 'utf8'),
        new Utils.Writer().writeUInt64LEBn(new BigNumber(changeAmount)).toArray(),
        Utils.toArray(JSON.stringify([{ name: 'label', value: label }]), 'utf8')
      ]
      const changeScript = await token.lock(changeFields, protocolID, changeKeyID, 'self', true, false)
      txOutputs.push({
        satoshis: 1,
        lockingScript: changeScript.toHex(),
        basket: 'mytokens',
        customInstructions: JSON.stringify({ protocolID, keyID: changeKeyID, counterparty: 'self' }),
        tags: ['mytokens', 'change']
      })
    }

    // 4. Phase 1: Create transaction
    const response = await wallet.createAction({
      description: `Send ${sendAmount} tokens`,
      inputBEEF: beef.toBinary(),
      inputs,
      outputs: txOutputs,
      options: { randomizeOutputs: false }
    })

    // 5. Phase 2: Sign token inputs manually
    const txToSign = Transaction.fromBEEF(response.signableTransaction!.tx)

    for (let i = 0; i < inputs.length; i++) {
      const cu = JSON.parse(customInstructions[i])
      txToSign.inputs[i].unlockingScriptTemplate = new PushDrop(wallet).unlock(
        cu.protocolID,
        cu.keyID,
        cu.counterparty
      )
    }

    await txToSign.sign()

    // 6. Phase 3: Submit signatures
    const spends: Record<string, { unlockingScript: string }> = {}
    for (let i = 0; i < inputs.length; i++) {
      spends[String(i)] = { unlockingScript: txToSign.inputs[i].unlockingScript!.toHex() }
    }

    const finalTx = await wallet.signAction({
      reference: response.signableTransaction!.reference,
      spends
    })

    // 7. Send via MessageBox
    const { publicKey: sender } = await wallet.getPublicKey({ identityKey: true })
    await messageBoxClient.sendMessage({
      recipient,
      messageBox: 'mytokenpayments',
      body: { tokenId, amount: sendAmount, transaction: finalTx.tx, keyID: recipientKeyID, protocolID, sender }
    })
  }

  // ... UI code ...
}
```

**Two-Phase Signing:**
1. `createAction()` - Wallet adds BSV inputs, estimates unlock length
2. Manual signing - Create PushDrop unlock scripts using `customInstructions`
3. `signAction()` - Submit completed unlocking scripts

> **Reference**: [BRC-62: BEEF](https://hub.bsvblockchain.org/brc/transactions/0062), [BRC-33: MessageBox](https://hub.bsvblockchain.org/brc/peer-to-peer/0033)

---

## 5. Token Receiving

`src/components/ReceiveTokens.tsx`:

```typescript
import { WalletClient, MessageBoxClient } from '@bsv/sdk'

export function ReceiveTokens({ wallet, messageBoxClient }: { wallet: WalletClient, messageBoxClient: MessageBoxClient }) {
  const acceptToken = async (message: any) => {
    await wallet.internalizeAction({
      tx: message.transaction,
      outputs: [{
        outputIndex: 0,
        protocol: 'basket insertion',
        insertionRemittance: { basket: 'mytokens' },
        customInstructions: JSON.stringify({
          protocolID: message.protocolID,
          keyID: message.keyID,
          counterparty: message.sender
        })
      }],
      description: 'Receive tokens'
    })

    await messageBoxClient.acknowledgeMessage({ messageIds: [message.messageId] })
  }

  const loadMessages = async () => {
    const result = await messageBoxClient.getMessages({ messageBox: 'mytokenpayments' })
    return result.messages
  }

  // ... UI code ...
}
```

**Pattern**: `internalizeAction` imports transaction with derivation params for future spending.

---

## Key Concepts

### Privacy (BRC-42)
Every token output uses a unique derived address:
```typescript
await token.lock(fields, [2, 'mytoken'], randomKeyID, counterparty, forSelf, false)
```
- Random `keyID` per output prevents address reuse
- Cannot link UTXOs to same owner

### BEEF Format (BRC-62)
Transactions include parent txs + merkle proofs for SPV validation without full blockchain.

### Critical: customInstructions
**MUST** store derivation params to spend tokens:
```typescript
customInstructions: JSON.stringify({ protocolID, keyID, counterparty })
```
Without this, tokens are **permanently unspendable**.

---

## Troubleshooting

- **"Failed to connect"** - Install/unlock BSV Desktop Wallet
- **"Insufficient balance"** - Need BSV for tx fees (get testnet coins from [faucet](https://faucet.bsvb.tech/))
- **Tokens not appearing** - Check correct basket name, verify `customInstructions` stored

---

## Resources

- [BRC-48: PushDrop](https://hub.bsvblockchain.org/brc/scripts/0048)
- [BRC-42: Key Derivation](https://hub.bsvblockchain.org/brc/key-derivation/0042)
- [BRC-62: BEEF](https://hub.bsvblockchain.org/brc/transactions/0062)
- [BRC-33: MessageBox](https://hub.bsvblockchain.org/brc/peer-to-peer/0033)
- [BRC-100: Wallet-to-Application Interface](https://hub.bsvblockchain.org/brc/wallet/0100)
- [BSV Desktop Wallet](https://desktop.bsvb.tech/)
