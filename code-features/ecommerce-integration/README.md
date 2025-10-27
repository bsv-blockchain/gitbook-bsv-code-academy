# E-commerce Payment Integration

Complete examples for integrating BSV payments into e-commerce platforms, including shopping carts, checkout flows, and payment verification.

## Overview

Integrating BSV into e-commerce systems enables fast, low-cost payments with immediate settlement. This guide covers payment request generation, checkout integration, payment verification, inventory management, and customer order fulfillment. These patterns work with any e-commerce platform or custom solution.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [P2PKH](../../sdk-components/p2pkh/README.md)
- [ARC](../../sdk-components/arc/README.md)
- [UTXO Management](../../sdk-components/utxo-management/README.md)

## Payment Request Generator

```typescript
import { PrivateKey, Script } from '@bsv/sdk'

/**
 * Payment Request Generator
 *
 * Generate payment requests for e-commerce checkout
 */
class PaymentRequestGenerator {
  private merchantAddress: string

  constructor(merchantKey: PrivateKey) {
    this.merchantAddress = merchantKey.toPublicKey().toAddress()
  }

  /**
   * Generate payment request for cart
   */
  generatePaymentRequest(params: {
    cartId: string
    items: CartItem[]
    customerId?: string
    customerEmail?: string
  }): PaymentRequest {
    try {
      // Calculate totals
      const subtotal = params.items.reduce((sum, item) => sum + (item.price * item.quantity), 0)
      const tax = Math.floor(subtotal * 0.08) // 8% tax
      const shipping = this.calculateShipping(params.items)
      const total = subtotal + tax + shipping

      const request: PaymentRequest = {
        requestId: this.generateRequestId(),
        cartId: params.cartId,
        merchantAddress: this.merchantAddress,
        amount: total,
        currency: 'satoshis',
        items: params.items,
        breakdown: {
          subtotal,
          tax,
          shipping,
          total
        },
        customerId: params.customerId,
        customerEmail: params.customerEmail,
        createdAt: Date.now(),
        expiresAt: Date.now() + 30 * 60 * 1000, // 30 minutes
        status: 'pending'
      }

      console.log('Payment request generated')
      console.log('Request ID:', request.requestId)
      console.log('Amount:', request.amount, 'satoshis')

      return request
    } catch (error) {
      throw new Error(`Payment request generation failed: ${error.message}`)
    }
  }

  /**
   * Generate payment URI (BIP-21 style)
   */
  generatePaymentURI(request: PaymentRequest): string {
    const uri = `bitcoin:${request.merchantAddress}?` +
      `amount=${request.amount / 100000000}&` + // Convert to BSV
      `label=${encodeURIComponent('E-commerce Payment')}&` +
      `message=${encodeURIComponent(`Order ${request.requestId}`)}`

    return uri
  }

  /**
   * Generate QR code data
   */
  generateQRCodeData(request: PaymentRequest): string {
    return this.generatePaymentURI(request)
  }

  /**
   * Calculate shipping cost
   */
  private calculateShipping(items: CartItem[]): number {
    const totalWeight = items.reduce((sum, item) => sum + (item.weight || 0) * item.quantity, 0)

    if (totalWeight === 0) return 0
    if (totalWeight < 1000) return 5000 // 5000 sats for < 1kg
    if (totalWeight < 5000) return 10000 // 10000 sats for < 5kg
    return 20000 // 20000 sats for > 5kg
  }

  /**
   * Generate unique request ID
   */
  private generateRequestId(): string {
    return `PAY-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface CartItem {
  itemId: string
  name: string
  price: number
  quantity: number
  weight?: number // grams
}

interface PaymentRequest {
  requestId: string
  cartId: string
  merchantAddress: string
  amount: number
  currency: string
  items: CartItem[]
  breakdown: {
    subtotal: number
    tax: number
    shipping: number
    total: number
  }
  customerId?: string
  customerEmail?: string
  createdAt: number
  expiresAt: number
  status: 'pending' | 'paid' | 'expired'
}

/**
 * Usage Example
 */
function paymentRequestExample() {
  const merchantKey = PrivateKey.fromRandom()
  const generator = new PaymentRequestGenerator(merchantKey)

  const request = generator.generatePaymentRequest({
    cartId: 'CART-123',
    items: [
      { itemId: 'ITEM-001', name: 'T-Shirt', price: 25000, quantity: 2, weight: 200 },
      { itemId: 'ITEM-002', name: 'Mug', price: 15000, quantity: 1, weight: 300 }
    ],
    customerEmail: 'customer@example.com'
  })

  console.log('Payment request:', request)

  // Generate payment URI
  const uri = generator.generatePaymentURI(request)
  console.log('Payment URI:', uri)

  // Generate QR code data
  const qrData = generator.generateQRCodeData(request)
  console.log('QR code data:', qrData)
}
```

## Checkout Integration

```typescript
import { Transaction, PrivateKey, P2PKH, Script, ARC } from '@bsv/sdk'

/**
 * Checkout Manager
 *
 * Manage checkout process and payment verification
 */
class CheckoutManager {
  private arc: ARC
  private merchantKey: PrivateKey
  private pendingPayments: Map<string, PendingCheckout> = new Map()

  constructor(merchantKey: PrivateKey, arcUrl: string = 'https://arc.taal.com') {
    this.merchantKey = merchantKey
    this.arc = new ARC(arcUrl)
  }

  /**
   * Initialize checkout session
   */
  initializeCheckout(params: {
    cartId: string
    items: CartItem[]
    customerInfo: CustomerInfo
  }): CheckoutSession {
    try {
      const sessionId = this.generateSessionId()

      // Calculate total
      const subtotal = params.items.reduce((sum, item) => sum + (item.price * item.quantity), 0)
      const total = subtotal + Math.floor(subtotal * 0.08) // with tax

      const session: CheckoutSession = {
        sessionId,
        cartId: params.cartId,
        items: params.items,
        customerInfo: params.customerInfo,
        total,
        merchantAddress: this.merchantKey.toPublicKey().toAddress(),
        status: 'initialized',
        createdAt: Date.now(),
        expiresAt: Date.now() + 30 * 60 * 1000 // 30 minutes
      }

      console.log('Checkout session initialized')
      console.log('Session ID:', sessionId)
      console.log('Total:', total, 'satoshis')

      return session
    } catch (error) {
      throw new Error(`Checkout initialization failed: ${error.message}`)
    }
  }

  /**
   * Process payment for checkout
   */
  async processPayment(params: {
    sessionId: string
    customerTx: Transaction
    waitForConfirmation?: boolean
  }): Promise<PaymentResult> {
    try {
      const session = this.pendingPayments.get(params.sessionId)
      if (!session) {
        throw new Error('Checkout session not found')
      }

      console.log('Processing payment for session:', params.sessionId)

      const txid = params.customerTx.id('hex')

      // Verify payment amount and recipient
      const verification = this.verifyPayment(
        params.customerTx,
        session.session.merchantAddress,
        session.session.total
      )

      if (!verification.valid) {
        return {
          success: false,
          sessionId: params.sessionId,
          reason: verification.reason,
          timestamp: Date.now()
        }
      }

      // Broadcast transaction
      try {
        await this.arc.broadcastTransaction(params.customerTx)
        console.log('Payment broadcasted:', txid)
      } catch (error) {
        return {
          success: false,
          sessionId: params.sessionId,
          reason: `Broadcast failed: ${error.message}`,
          timestamp: Date.now()
        }
      }

      // Wait for confirmation if required
      if (params.waitForConfirmation) {
        try {
          await this.waitForConfirmation(txid, 1, 60000) // 1 conf, 60s timeout
          console.log('Payment confirmed')
        } catch (error) {
          return {
            success: false,
            sessionId: params.sessionId,
            reason: 'Confirmation timeout',
            timestamp: Date.now()
          }
        }
      }

      // Update session
      session.paymentTxid = txid
      session.paidAt = Date.now()
      session.session.status = 'paid'

      // Create order
      const order = await this.createOrder(session)

      return {
        success: true,
        sessionId: params.sessionId,
        txid,
        orderId: order.orderId,
        timestamp: Date.now()
      }
    } catch (error) {
      return {
        success: false,
        sessionId: params.sessionId,
        reason: error.message,
        timestamp: Date.now()
      }
    }
  }

  /**
   * Verify payment matches requirements
   */
  private verifyPayment(
    tx: Transaction,
    expectedAddress: string,
    expectedAmount: number
  ): { valid: boolean; reason?: string } {
    let totalPaid = 0

    for (const output of tx.outputs) {
      try {
        const outputAddress = output.lockingScript.toAddress()
        if (outputAddress === expectedAddress) {
          totalPaid += output.satoshis
        }
      } catch {
        continue
      }
    }

    if (totalPaid < expectedAmount) {
      return {
        valid: false,
        reason: `Insufficient payment: ${totalPaid} < ${expectedAmount}`
      }
    }

    return { valid: true }
  }

  /**
   * Wait for transaction confirmation
   */
  private async waitForConfirmation(
    txid: string,
    required: number,
    timeout: number
  ): Promise<void> {
    const startTime = Date.now()

    while (Date.now() - startTime < timeout) {
      try {
        const status = await this.arc.getTransactionStatus(txid)
        if ((status.confirmations || 0) >= required) {
          return
        }
      } catch (error) {
        // Continue waiting
      }

      await new Promise(resolve => setTimeout(resolve, 5000))
    }

    throw new Error('Confirmation timeout')
  }

  /**
   * Create order from checkout session
   */
  private async createOrder(checkout: PendingCheckout): Promise<Order> {
    const order: Order = {
      orderId: this.generateOrderId(),
      sessionId: checkout.session.sessionId,
      items: checkout.session.items,
      customerInfo: checkout.session.customerInfo,
      total: checkout.session.total,
      paymentTxid: checkout.paymentTxid!,
      status: 'confirmed',
      createdAt: Date.now()
    }

    console.log('Order created:', order.orderId)

    return order
  }

  /**
   * Generate session ID
   */
  private generateSessionId(): string {
    return `SESSION-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }

  /**
   * Generate order ID
   */
  private generateOrderId(): string {
    return `ORDER-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface CheckoutSession {
  sessionId: string
  cartId: string
  items: CartItem[]
  customerInfo: CustomerInfo
  total: number
  merchantAddress: string
  status: 'initialized' | 'paid' | 'expired'
  createdAt: number
  expiresAt: number
}

interface CustomerInfo {
  email: string
  name: string
  shippingAddress?: {
    street: string
    city: string
    state: string
    zip: string
    country: string
  }
}

interface PendingCheckout {
  session: CheckoutSession
  paymentTxid?: string
  paidAt?: number
}

interface PaymentResult {
  success: boolean
  sessionId: string
  txid?: string
  orderId?: string
  reason?: string
  timestamp: number
}

interface Order {
  orderId: string
  sessionId: string
  items: CartItem[]
  customerInfo: CustomerInfo
  total: number
  paymentTxid: string
  status: 'confirmed' | 'processing' | 'shipped' | 'delivered'
  createdAt: number
  shippedAt?: number
  deliveredAt?: number
}

/**
 * Usage Example
 */
async function checkoutExample() {
  const merchantKey = PrivateKey.fromRandom()
  const checkoutManager = new CheckoutManager(merchantKey)

  // Initialize checkout
  const session = checkoutManager.initializeCheckout({
    cartId: 'CART-123',
    items: [
      { itemId: 'ITEM-001', name: 'Product 1', price: 50000, quantity: 2 },
      { itemId: 'ITEM-002', name: 'Product 2', price: 30000, quantity: 1 }
    ],
    customerInfo: {
      email: 'customer@example.com',
      name: 'John Doe',
      shippingAddress: {
        street: '123 Main St',
        city: 'New York',
        state: 'NY',
        zip: '10001',
        country: 'USA'
      }
    }
  })

  console.log('Checkout session:', session)

  // Customer creates and sends payment transaction
  const customerTx = new Transaction()
  // ... customer builds and signs transaction ...

  // Process payment
  const result = await checkoutManager.processPayment({
    sessionId: session.sessionId,
    customerTx,
    waitForConfirmation: true
  })

  if (result.success) {
    console.log('Payment successful!')
    console.log('Order ID:', result.orderId)
  } else {
    console.error('Payment failed:', result.reason)
  }
}
```

## Order Management System

```typescript
/**
 * Order Management System
 *
 * Complete order lifecycle management
 */
class OrderManagementSystem {
  private orders: Map<string, EcommerceOrder> = new Map()

  /**
   * Create new order
   */
  createOrder(params: {
    customerId: string
    customerEmail: string
    items: OrderItem[]
    shippingAddress: Address
    paymentTxid: string
    total: number
  }): EcommerceOrder {
    try {
      const orderId = this.generateOrderId()

      const order: EcommerceOrder = {
        orderId,
        customerId: params.customerId,
        customerEmail: params.customerEmail,
        items: params.items,
        shippingAddress: params.shippingAddress,
        paymentTxid: params.paymentTxid,
        total: params.total,
        status: 'pending',
        createdAt: Date.now(),
        history: [
          {
            status: 'pending',
            timestamp: Date.now(),
            note: 'Order created'
          }
        ]
      }

      this.orders.set(orderId, order)

      console.log('Order created:', orderId)

      return order
    } catch (error) {
      throw new Error(`Order creation failed: ${error.message}`)
    }
  }

  /**
   * Update order status
   */
  updateOrderStatus(
    orderId: string,
    status: OrderStatus,
    note?: string
  ): EcommerceOrder {
    const order = this.orders.get(orderId)
    if (!order) {
      throw new Error('Order not found')
    }

    order.status = status
    order.history.push({
      status,
      timestamp: Date.now(),
      note
    })

    console.log(`Order ${orderId} status updated to ${status}`)

    return order
  }

  /**
   * Mark order as shipped
   */
  markAsShipped(
    orderId: string,
    trackingNumber: string,
    carrier: string
  ): EcommerceOrder {
    const order = this.orders.get(orderId)
    if (!order) {
      throw new Error('Order not found')
    }

    order.status = 'shipped'
    order.trackingNumber = trackingNumber
    order.carrier = carrier
    order.shippedAt = Date.now()
    order.history.push({
      status: 'shipped',
      timestamp: Date.now(),
      note: `Shipped via ${carrier}, tracking: ${trackingNumber}`
    })

    console.log(`Order ${orderId} marked as shipped`)

    return order
  }

  /**
   * Mark order as delivered
   */
  markAsDelivered(orderId: string): EcommerceOrder {
    const order = this.orders.get(orderId)
    if (!order) {
      throw new Error('Order not found')
    }

    order.status = 'delivered'
    order.deliveredAt = Date.now()
    order.history.push({
      status: 'delivered',
      timestamp: Date.now(),
      note: 'Order delivered'
    })

    console.log(`Order ${orderId} marked as delivered`)

    return order
  }

  /**
   * Process refund
   */
  async processRefund(
    orderId: string,
    refundAmount: number,
    reason: string,
    merchantKey: PrivateKey,
    customerAddress: string,
    utxo: any
  ): Promise<{ order: EcommerceOrder; refundTx: Transaction }> {
    try {
      const order = this.orders.get(orderId)
      if (!order) {
        throw new Error('Order not found')
      }

      console.log('Processing refund for order:', orderId)
      console.log('Refund amount:', refundAmount)

      // Create refund transaction
      const refundTx = new Transaction()

      refundTx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(merchantKey),
        sequence: 0xffffffff
      })

      refundTx.addOutput({
        satoshis: refundAmount,
        lockingScript: Script.fromAddress(customerAddress)
      })

      const fee = 500
      const change = utxo.satoshis - refundAmount - fee

      if (change >= 546) {
        refundTx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(merchantKey.toPublicKey().toHash())
        })
      }

      await refundTx.sign()

      // Update order
      order.status = 'refunded'
      order.refundTxid = refundTx.id('hex')
      order.refundAmount = refundAmount
      order.refundReason = reason
      order.refundedAt = Date.now()
      order.history.push({
        status: 'refunded',
        timestamp: Date.now(),
        note: `Refund processed: ${refundAmount} sats. Reason: ${reason}`
      })

      console.log('Refund processed:', refundTx.id('hex'))

      return { order, refundTx }
    } catch (error) {
      throw new Error(`Refund processing failed: ${error.message}`)
    }
  }

  /**
   * Get order by ID
   */
  getOrder(orderId: string): EcommerceOrder | undefined {
    return this.orders.get(orderId)
  }

  /**
   * Get orders by customer
   */
  getOrdersByCustomer(customerId: string): EcommerceOrder[] {
    return Array.from(this.orders.values()).filter(o => o.customerId === customerId)
  }

  /**
   * Get orders by status
   */
  getOrdersByStatus(status: OrderStatus): EcommerceOrder[] {
    return Array.from(this.orders.values()).filter(o => o.status === status)
  }

  /**
   * Generate order ID
   */
  private generateOrderId(): string {
    const timestamp = Date.now().toString(36)
    const random = Math.random().toString(36).substr(2, 5)
    return `ORD-${timestamp}-${random}`.toUpperCase()
  }
}

type OrderStatus = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled' | 'refunded'

interface OrderItem {
  itemId: string
  name: string
  price: number
  quantity: number
  sku?: string
}

interface Address {
  street: string
  city: string
  state: string
  zip: string
  country: string
}

interface EcommerceOrder {
  orderId: string
  customerId: string
  customerEmail: string
  items: OrderItem[]
  shippingAddress: Address
  paymentTxid: string
  total: number
  status: OrderStatus
  createdAt: number
  shippedAt?: number
  deliveredAt?: number
  refundedAt?: number
  trackingNumber?: string
  carrier?: string
  refundTxid?: string
  refundAmount?: number
  refundReason?: string
  history: OrderHistoryEntry[]
}

interface OrderHistoryEntry {
  status: OrderStatus
  timestamp: number
  note?: string
}

/**
 * Usage Example
 */
async function orderManagementExample() {
  const oms = new OrderManagementSystem()

  // Create order
  const order = oms.createOrder({
    customerId: 'CUST-001',
    customerEmail: 'customer@example.com',
    items: [
      { itemId: 'ITEM-001', name: 'Product 1', price: 50000, quantity: 1, sku: 'SKU-001' }
    ],
    shippingAddress: {
      street: '123 Main St',
      city: 'New York',
      state: 'NY',
      zip: '10001',
      country: 'USA'
    },
    paymentTxid: 'payment-tx-id...',
    total: 50000
  })

  console.log('Order created:', order)

  // Update status
  oms.updateOrderStatus(order.orderId, 'processing', 'Payment confirmed')

  // Mark as shipped
  oms.markAsShipped(order.orderId, 'TRACK-123456', 'UPS')

  // Mark as delivered
  oms.markAsDelivered(order.orderId)

  // Get customer orders
  const customerOrders = oms.getOrdersByCustomer('CUST-001')
  console.log('Customer orders:', customerOrders.length)
}
```

## Related Examples

- [Payment Processing](../payment-processing/README.md)
- [Payment Distribution](../payment-distribution/README.md)
- [Double-Spend Detection](../double-spend-detection/README.md)
- [Marketplace](../marketplace/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [P2PKH](../../sdk-components/p2pkh/README.md) - Standard payments
- [ARC](../../sdk-components/arc/README.md) - Transaction broadcasting
- [UTXO Management](../../sdk-components/utxo-management/README.md) - UTXO handling

**Learning Paths:**
- [E-commerce Integration](../../learning-paths/intermediate/ecommerce-integration/README.md)
- [Payment Gateways](../../learning-paths/advanced/payment-gateways/README.md)
- [Business Applications](../../learning-paths/advanced/business-applications/README.md)
