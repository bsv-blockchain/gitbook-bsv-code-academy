# Marketplace Escrow and Payment Patterns

Complete examples for implementing marketplace functionality with escrow, dispute resolution, and secure payment patterns.

## Overview

Marketplace applications require secure payment handling, escrow mechanisms, and dispute resolution. BSV's programmable scripts enable trustless escrow, atomic exchanges, and sophisticated marketplace logic. This guide covers escrow creation, buyer-seller interactions, refunds, and dispute management.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Script](../../sdk-components/script/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)
- [Multi-Signature](../../sdk-components/signatures/README.md)

## Basic Escrow System

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP, P2PKH } from '@bsv/sdk'

/**
 * Escrow Manager
 *
 * Manage escrow transactions for marketplace transactions
 */
class EscrowManager {
  /**
   * Create escrow transaction
   *
   * Funds locked until buyer and seller both sign (2-of-2) or arbiter releases
   */
  async createEscrow(params: {
    buyerKey: PrivateKey
    sellerPubKey: PublicKey
    arbiterPubKey: PublicKey
    amount: number
    orderId: string
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<{
    escrowTx: Transaction
    escrow: Escrow
  }> {
    try {
      console.log('Creating escrow for order:', params.orderId)
      console.log('Amount:', params.amount)

      const escrowTx = new Transaction()

      // Add buyer's input
      escrowTx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.buyerKey),
        sequence: 0xffffffff
      })

      // Create 2-of-3 multisig escrow script
      const escrowScript = this.createEscrowScript(
        params.buyerKey.toPublicKey(),
        params.sellerPubKey,
        params.arbiterPubKey
      )

      // Add escrow output
      escrowTx.addOutput({
        satoshis: params.amount,
        lockingScript: escrowScript
      })

      // Add change output
      const fee = 1000
      const change = params.utxo.satoshis - params.amount - fee

      if (change >= 546) {
        escrowTx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.buyerKey.toPublicKey().toHash())
        })
      }

      await escrowTx.sign()

      const escrow: Escrow = {
        orderId: params.orderId,
        escrowTxid: escrowTx.id('hex'),
        amount: params.amount,
        buyerAddress: params.buyerKey.toPublicKey().toAddress(),
        sellerAddress: params.sellerPubKey.toAddress(),
        arbiterAddress: params.arbiterPubKey.toAddress(),
        status: 'locked',
        createdAt: Date.now()
      }

      console.log('Escrow created')
      console.log('Escrow TXID:', escrow.escrowTxid)

      return { escrowTx, escrow }
    } catch (error) {
      throw new Error(`Escrow creation failed: ${error.message}`)
    }
  }

  /**
   * Release escrow to seller (successful transaction)
   */
  async releaseToSeller(
    escrow: Escrow,
    buyerKey: PrivateKey,
    sellerKey: PrivateKey,
    escrowUTXO: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    }
  ): Promise<Transaction> {
    try {
      console.log('Releasing escrow to seller:', escrow.orderId)

      const releaseTx = new Transaction()

      releaseTx.addInput({
        sourceTXID: escrowUTXO.txid,
        sourceOutputIndex: escrowUTXO.vout,
        unlockingScript: new Script(),
        sequence: 0xffffffff
      })

      // Pay seller
      const fee = 500
      const sellerAmount = escrowUTXO.satoshis - fee

      releaseTx.addOutput({
        satoshis: sellerAmount,
        lockingScript: Script.fromAddress(escrow.sellerAddress)
      })

      // Create 2-of-3 multisig unlocking script
      // Both buyer and seller sign
      const input = releaseTx.inputs[0]
      const preimage = input.getPreimage(escrowUTXO.lockingScript)

      const buyerSig = buyerKey.sign(preimage)
      const sellerSig = sellerKey.sign(preimage)

      // Build unlocking script: OP_0 <sig1> <sig2>
      const unlockingScript = new Script()
      unlockingScript.writeOpCode(OP.OP_0) // OP_CHECKMULTISIG bug
      unlockingScript.writeBin(buyerSig.toDER())
      unlockingScript.writeBin(sellerSig.toDER())

      releaseTx.inputs[0].unlockingScript = unlockingScript

      console.log('Escrow released to seller')
      console.log('Release TXID:', releaseTx.id('hex'))

      escrow.status = 'released'
      escrow.releasedAt = Date.now()
      escrow.releaseTxid = releaseTx.id('hex')

      return releaseTx
    } catch (error) {
      throw new Error(`Escrow release failed: ${error.message}`)
    }
  }

  /**
   * Refund escrow to buyer (cancelled or disputed)
   */
  async refundToBuyer(
    escrow: Escrow,
    buyerKey: PrivateKey,
    arbiterKey: PrivateKey,
    escrowUTXO: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    }
  ): Promise<Transaction> {
    try {
      console.log('Refunding escrow to buyer:', escrow.orderId)

      const refundTx = new Transaction()

      refundTx.addInput({
        sourceTXID: escrowUTXO.txid,
        sourceOutputIndex: escrowUTXO.vout,
        unlockingScript: new Script(),
        sequence: 0xffffffff
      })

      // Refund to buyer
      const fee = 500
      const refundAmount = escrowUTXO.satoshis - fee

      refundTx.addOutput({
        satoshis: refundAmount,
        lockingScript: Script.fromAddress(escrow.buyerAddress)
      })

      // Create unlocking script with buyer and arbiter signatures
      const input = refundTx.inputs[0]
      const preimage = input.getPreimage(escrowUTXO.lockingScript)

      const buyerSig = buyerKey.sign(preimage)
      const arbiterSig = arbiterKey.sign(preimage)

      const unlockingScript = new Script()
      unlockingScript.writeOpCode(OP.OP_0)
      unlockingScript.writeBin(buyerSig.toDER())
      unlockingScript.writeBin(arbiterSig.toDER())

      refundTx.inputs[0].unlockingScript = unlockingScript

      console.log('Escrow refunded to buyer')
      console.log('Refund TXID:', refundTx.id('hex'))

      escrow.status = 'refunded'
      escrow.refundedAt = Date.now()
      escrow.refundTxid = refundTx.id('hex')

      return refundTx
    } catch (error) {
      throw new Error(`Escrow refund failed: ${error.message}`)
    }
  }

  /**
   * Create 2-of-3 multisig escrow script
   */
  private createEscrowScript(
    buyerPubKey: PublicKey,
    sellerPubKey: PublicKey,
    arbiterPubKey: PublicKey
  ): Script {
    const script = new Script()

    // 2-of-3 multisig: 2 <pubkey1> <pubkey2> <pubkey3> 3 OP_CHECKMULTISIG
    script.writeOpCode(OP.OP_2)
    script.writeBin(buyerPubKey.encode())
    script.writeBin(sellerPubKey.encode())
    script.writeBin(arbiterPubKey.encode())
    script.writeOpCode(OP.OP_3)
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }
}

interface Escrow {
  orderId: string
  escrowTxid: string
  amount: number
  buyerAddress: string
  sellerAddress: string
  arbiterAddress: string
  status: 'locked' | 'released' | 'refunded' | 'disputed'
  createdAt: number
  releasedAt?: number
  refundedAt?: number
  releaseTxid?: string
  refundTxid?: string
}

/**
 * Usage Example
 */
async function escrowExample() {
  const escrowManager = new EscrowManager()

  const buyer = PrivateKey.fromRandom()
  const seller = PrivateKey.fromRandom()
  const arbiter = PrivateKey.fromRandom()

  const buyerUTXO = {
    txid: 'buyer-utxo...',
    vout: 0,
    satoshis: 200000,
    script: new P2PKH().lock(buyer.toPublicKey().toHash())
  }

  // Create escrow
  const { escrowTx, escrow } = await escrowManager.createEscrow({
    buyerKey: buyer,
    sellerPubKey: seller.toPublicKey(),
    arbiterPubKey: arbiter.toPublicKey(),
    amount: 100000,
    orderId: 'ORDER-001',
    utxo: buyerUTXO
  })

  console.log('Escrow created:', escrow)

  // Later: Release to seller
  const escrowUTXO = {
    txid: escrowTx.id('hex'),
    vout: 0,
    satoshis: 100000,
    lockingScript: escrowTx.outputs[0].lockingScript
  }

  const releaseTx = await escrowManager.releaseToSeller(
    escrow,
    buyer,
    seller,
    escrowUTXO
  )

  console.log('Released to seller:', releaseTx.id('hex'))
}
```

## Marketplace Order Manager

```typescript
import { Transaction, PrivateKey, PublicKey } from '@bsv/sdk'

/**
 * Marketplace Order Manager
 *
 * Manage complete order lifecycle
 */
class OrderManager {
  private orders: Map<string, MarketplaceOrder> = new Map()
  private escrowManager: EscrowManager

  constructor() {
    this.escrowManager = new EscrowManager()
  }

  /**
   * Create new marketplace order
   */
  async createOrder(params: {
    buyerKey: PrivateKey
    sellerAddress: string
    arbiterPubKey: PublicKey
    itemId: string
    itemName: string
    price: number
    utxo: any
  }): Promise<MarketplaceOrder> {
    try {
      const orderId = this.generateOrderId()

      console.log('Creating order:', orderId)
      console.log('Item:', params.itemName)
      console.log('Price:', params.price)

      // Create escrow
      const sellerPubKey = this.addressToPublicKey(params.sellerAddress)
      const { escrowTx, escrow } = await this.escrowManager.createEscrow({
        buyerKey: params.buyerKey,
        sellerPubKey,
        arbiterPubKey: params.arbiterPubKey,
        amount: params.price,
        orderId,
        utxo: params.utxo
      })

      const order: MarketplaceOrder = {
        orderId,
        itemId: params.itemId,
        itemName: params.itemName,
        price: params.price,
        buyerAddress: params.buyerKey.toPublicKey().toAddress(),
        sellerAddress: params.sellerAddress,
        status: 'pending',
        escrow,
        createdAt: Date.now(),
        events: [
          {
            type: 'order_created',
            timestamp: Date.now(),
            data: { orderId, price: params.price }
          }
        ]
      }

      this.orders.set(orderId, order)

      console.log('Order created successfully')

      return order
    } catch (error) {
      throw new Error(`Order creation failed: ${error.message}`)
    }
  }

  /**
   * Mark order as shipped
   */
  markAsShipped(
    orderId: string,
    trackingNumber: string
  ): MarketplaceOrder {
    const order = this.orders.get(orderId)
    if (!order) {
      throw new Error('Order not found')
    }

    order.status = 'shipped'
    order.trackingNumber = trackingNumber
    order.shippedAt = Date.now()
    order.events.push({
      type: 'order_shipped',
      timestamp: Date.now(),
      data: { trackingNumber }
    })

    console.log('Order marked as shipped:', orderId)

    return order
  }

  /**
   * Complete order and release payment
   */
  async completeOrder(
    orderId: string,
    buyerKey: PrivateKey,
    sellerKey: PrivateKey,
    escrowUTXO: any
  ): Promise<MarketplaceOrder> {
    try {
      const order = this.orders.get(orderId)
      if (!order) {
        throw new Error('Order not found')
      }

      console.log('Completing order:', orderId)

      // Release escrow to seller
      const releaseTx = await this.escrowManager.releaseToSeller(
        order.escrow,
        buyerKey,
        sellerKey,
        escrowUTXO
      )

      order.status = 'completed'
      order.completedAt = Date.now()
      order.releaseTxid = releaseTx.id('hex')
      order.events.push({
        type: 'order_completed',
        timestamp: Date.now(),
        data: { releaseTxid: releaseTx.id('hex') }
      })

      console.log('Order completed successfully')

      return order
    } catch (error) {
      throw new Error(`Order completion failed: ${error.message}`)
    }
  }

  /**
   * Cancel order and refund buyer
   */
  async cancelOrder(
    orderId: string,
    reason: string,
    buyerKey: PrivateKey,
    arbiterKey: PrivateKey,
    escrowUTXO: any
  ): Promise<MarketplaceOrder> {
    try {
      const order = this.orders.get(orderId)
      if (!order) {
        throw new Error('Order not found')
      }

      console.log('Cancelling order:', orderId)
      console.log('Reason:', reason)

      // Refund escrow to buyer
      const refundTx = await this.escrowManager.refundToBuyer(
        order.escrow,
        buyerKey,
        arbiterKey,
        escrowUTXO
      )

      order.status = 'cancelled'
      order.cancelledAt = Date.now()
      order.cancellationReason = reason
      order.refundTxid = refundTx.id('hex')
      order.events.push({
        type: 'order_cancelled',
        timestamp: Date.now(),
        data: { reason, refundTxid: refundTx.id('hex') }
      })

      console.log('Order cancelled successfully')

      return order
    } catch (error) {
      throw new Error(`Order cancellation failed: ${error.message}`)
    }
  }

  /**
   * Open dispute
   */
  openDispute(
    orderId: string,
    disputeReason: string,
    disputedBy: 'buyer' | 'seller'
  ): MarketplaceOrder {
    const order = this.orders.get(orderId)
    if (!order) {
      throw new Error('Order not found')
    }

    order.status = 'disputed'
    order.dispute = {
      reason: disputeReason,
      disputedBy,
      openedAt: Date.now(),
      status: 'open'
    }
    order.events.push({
      type: 'dispute_opened',
      timestamp: Date.now(),
      data: { reason: disputeReason, by: disputedBy }
    })

    console.log('Dispute opened for order:', orderId)

    return order
  }

  /**
   * Resolve dispute
   */
  resolveDispute(
    orderId: string,
    resolution: 'refund_buyer' | 'pay_seller',
    arbiterNotes: string
  ): MarketplaceOrder {
    const order = this.orders.get(orderId)
    if (!order || !order.dispute) {
      throw new Error('Order or dispute not found')
    }

    order.dispute.status = 'resolved'
    order.dispute.resolution = resolution
    order.dispute.arbiterNotes = arbiterNotes
    order.dispute.resolvedAt = Date.now()
    order.events.push({
      type: 'dispute_resolved',
      timestamp: Date.now(),
      data: { resolution, arbiterNotes }
    })

    console.log('Dispute resolved:', resolution)

    return order
  }

  /**
   * Get order by ID
   */
  getOrder(orderId: string): MarketplaceOrder | undefined {
    return this.orders.get(orderId)
  }

  /**
   * Get all orders
   */
  getAllOrders(): MarketplaceOrder[] {
    return Array.from(this.orders.values())
  }

  /**
   * Get orders by status
   */
  getOrdersByStatus(status: OrderStatus): MarketplaceOrder[] {
    return Array.from(this.orders.values()).filter(o => o.status === status)
  }

  /**
   * Generate unique order ID
   */
  private generateOrderId(): string {
    return `ORDER-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }

  /**
   * Convert address to public key (simplified)
   */
  private addressToPublicKey(address: string): PublicKey {
    // In production, this would look up the actual public key
    // For demo purposes, we'll create a random one
    return PrivateKey.fromRandom().toPublicKey()
  }
}

type OrderStatus = 'pending' | 'shipped' | 'completed' | 'cancelled' | 'disputed'

interface MarketplaceOrder {
  orderId: string
  itemId: string
  itemName: string
  price: number
  buyerAddress: string
  sellerAddress: string
  status: OrderStatus
  escrow: Escrow
  createdAt: number
  shippedAt?: number
  completedAt?: number
  cancelledAt?: number
  trackingNumber?: string
  cancellationReason?: string
  releaseTxid?: string
  refundTxid?: string
  dispute?: Dispute
  events: OrderEvent[]
}

interface Dispute {
  reason: string
  disputedBy: 'buyer' | 'seller'
  openedAt: number
  resolvedAt?: number
  status: 'open' | 'resolved'
  resolution?: 'refund_buyer' | 'pay_seller'
  arbiterNotes?: string
}

interface OrderEvent {
  type: string
  timestamp: number
  data: any
}

/**
 * Usage Example
 */
async function marketplaceExample() {
  const orderManager = new OrderManager()

  const buyer = PrivateKey.fromRandom()
  const arbiter = PrivateKey.fromRandom()

  // Create order
  const order = await orderManager.createOrder({
    buyerKey: buyer,
    sellerAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    arbiterPubKey: arbiter.toPublicKey(),
    itemId: 'ITEM-001',
    itemName: 'Laptop Computer',
    price: 500000,
    utxo: {/* ... */}
  })

  console.log('Order created:', order.orderId)

  // Mark as shipped
  orderManager.markAsShipped(order.orderId, 'TRACK-123456')

  // Get orders by status
  const pendingOrders = orderManager.getOrdersByStatus('pending')
  console.log('Pending orders:', pendingOrders.length)
}
```

## Atomic Swap Marketplace

```typescript
import { Transaction, PrivateKey, Script, OP } from '@bsv/sdk'

/**
 * Atomic Swap Manager
 *
 * Trustless peer-to-peer item exchange
 */
class AtomicSwapManager {
  /**
   * Create atomic swap offer
   *
   * Alice offers item A for item B
   */
  async createSwapOffer(params: {
    offerorKey: PrivateKey
    offerorItemUTXO: any
    acceptorPubKey: PublicKey
    acceptorItemHash: string // Hash of item B
    timeout: number
  }): Promise<{
    offerTx: Transaction
    secret: string
    secretHash: string
  }> {
    try {
      const secret = this.generateSecret()
      const secretHash = this.hashSecret(secret)

      console.log('Creating atomic swap offer')
      console.log('Secret hash:', secretHash)

      const offerTx = new Transaction()

      // Add offeror's item
      offerTx.addInput({
        sourceTXID: params.offerorItemUTXO.txid,
        sourceOutputIndex: params.offerorItemUTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.offerorKey),
        sequence: 0xffffffff
      })

      // Create HTLC (Hash Time Locked Contract) output
      const htlcScript = this.createHTLCScript(
        secretHash,
        params.acceptorPubKey,
        params.offerorKey.toPublicKey(),
        params.timeout
      )

      offerTx.addOutput({
        satoshis: params.offerorItemUTXO.satoshis,
        lockingScript: htlcScript
      })

      await offerTx.sign()

      console.log('Swap offer created')
      console.log('Offer TXID:', offerTx.id('hex'))

      return {
        offerTx,
        secret,
        secretHash
      }
    } catch (error) {
      throw new Error(`Swap offer creation failed: ${error.message}`)
    }
  }

  /**
   * Accept swap offer
   *
   * Bob accepts by providing his item and the secret
   */
  async acceptSwap(params: {
    acceptorKey: PrivateKey
    acceptorItemUTXO: any
    offerTxid: string
    secret: string
    offerorAddress: string
  }): Promise<Transaction> {
    try {
      console.log('Accepting atomic swap')

      const acceptTx = new Transaction()

      // Add acceptor's item
      acceptTx.addInput({
        sourceTXID: params.acceptorItemUTXO.txid,
        sourceOutputIndex: params.acceptorItemUTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.acceptorKey),
        sequence: 0xffffffff
      })

      // Pay to offeror
      acceptTx.addOutput({
        satoshis: params.acceptorItemUTXO.satoshis,
        lockingScript: Script.fromAddress(params.offerorAddress)
      })

      await acceptTx.sign()

      console.log('Swap accepted')
      console.log('Accept TXID:', acceptTx.id('hex'))

      return acceptTx
    } catch (error) {
      throw new Error(`Swap acceptance failed: ${error.message}`)
    }
  }

  /**
   * Create Hash Time Locked Contract script
   */
  private createHTLCScript(
    secretHash: string,
    acceptorPubKey: PublicKey,
    offerorPubKey: PublicKey,
    timeout: number
  ): Script {
    const script = new Script()

    // IF secret matches
    //   <acceptorPubKey> OP_CHECKSIG
    // ELSE
    //   <timeout> OP_CHECKLOCKTIMEVERIFY OP_DROP
    //   <offerorPubKey> OP_CHECKSIG
    // ENDIF

    script.writeOpCode(OP.OP_IF)

    // Success path: reveal secret
    script.writeOpCode(OP.OP_HASH256)
    script.writeBin(Buffer.from(secretHash, 'hex'))
    script.writeOpCode(OP.OP_EQUALVERIFY)
    script.writeBin(acceptorPubKey.encode())
    script.writeOpCode(OP.OP_CHECKSIG)

    script.writeOpCode(OP.OP_ELSE)

    // Timeout path: refund to offeror
    script.writeBin(this.numberToBuffer(timeout))
    script.writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
    script.writeOpCode(OP.OP_DROP)
    script.writeBin(offerorPubKey.encode())
    script.writeOpCode(OP.OP_CHECKSIG)

    script.writeOpCode(OP.OP_ENDIF)

    return script
  }

  /**
   * Generate random secret
   */
  private generateSecret(): string {
    return Math.random().toString(36).substring(2) + Date.now().toString(36)
  }

  /**
   * Hash secret
   */
  private hashSecret(secret: string): string {
    const crypto = require('crypto')
    const hash = crypto.createHash('sha256').update(secret).digest()
    return crypto.createHash('sha256').update(hash).digest('hex')
  }

  /**
   * Convert number to buffer
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])
    const bytes: number[] = []
    let n = num
    while (n > 0) {
      bytes.push(n & 0xff)
      n >>= 8
    }
    return Buffer.from(bytes)
  }
}

/**
 * Usage Example
 */
async function atomicSwapExample() {
  const swapManager = new AtomicSwapManager()

  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()

  // Alice creates swap offer
  const { offerTx, secret, secretHash } = await swapManager.createSwapOffer({
    offerorKey: alice,
    offerorItemUTXO: {/* ... */},
    acceptorPubKey: bob.toPublicKey(),
    acceptorItemHash: 'item-b-hash',
    timeout: Math.floor(Date.now() / 1000) + 3600 // 1 hour
  })

  console.log('Swap offer:', offerTx.id('hex'))

  // Bob accepts swap
  const acceptTx = await swapManager.acceptSwap({
    acceptorKey: bob,
    acceptorItemUTXO: {/* ... */},
    offerTxid: offerTx.id('hex'),
    secret,
    offerorAddress: alice.toPublicKey().toAddress()
  })

  console.log('Swap completed:', acceptTx.id('hex'))
}
```

## Related Examples

- [Atomic Swaps](../atomic-swaps/README.md)
- [Payment Processing](../payment-processing/README.md)
- [Multi-Signature](../multi-signature/README.md)
- [Smart Contracts](../smart-contracts/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [Script](../../sdk-components/script/README.md) - Bitcoin script operations
- [Script Templates](../../sdk-components/script-templates/README.md) - Template system
- [Signatures](../../sdk-components/signatures/README.md) - Digital signatures

**Learning Paths:**
- [Escrow Patterns](../../learning-paths/intermediate/escrow-patterns/README.md)
- [Marketplace Development](../../learning-paths/advanced/marketplace-development/README.md)
- [Atomic Swaps](../../learning-paths/advanced/atomic-swaps/README.md)
