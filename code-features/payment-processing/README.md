# Payment Processing

Complete examples for implementing payment processing patterns including invoicing, recurring payments, and subscription management on BSV.

## Overview

Payment processing on BSV goes beyond simple transactions. This guide covers invoice generation, payment verification, recurring payments, subscription management, and payment workflow automation. These patterns are essential for building commercial applications on BSV.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [P2PKH](../../sdk-components/p2pkh/README.md)
- [ARC](../../sdk-components/arc/README.md)
- [UTXO Management](../../sdk-components/utxo-management/README.md)

## Invoice Generation and Payment

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Invoice Payment Processor
 *
 * Generate and process invoice payments
 */
class InvoiceProcessor {
  /**
   * Generate a payment invoice
   */
  generateInvoice(params: {
    merchantAddress: string
    amount: number
    invoiceId: string
    description: string
    expiresAt?: number
  }): Invoice {
    const invoice: Invoice = {
      id: params.invoiceId,
      merchantAddress: params.merchantAddress,
      amount: params.amount,
      description: params.description,
      createdAt: Date.now(),
      expiresAt: params.expiresAt || Date.now() + 24 * 60 * 60 * 1000, // 24 hours default
      status: 'pending',
      paymentTxid: null
    }

    console.log('Invoice generated')
    console.log('Invoice ID:', invoice.id)
    console.log('Amount:', invoice.amount, 'satoshis')
    console.log('Expires:', new Date(invoice.expiresAt).toISOString())

    return invoice
  }

  /**
   * Process invoice payment
   */
  async processInvoicePayment(
    invoice: Invoice,
    payerKey: PrivateKey,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    invoice: Invoice
  }> {
    try {
      // Validate invoice
      if (invoice.status !== 'pending') {
        throw new Error('Invoice already paid or expired')
      }

      if (Date.now() > invoice.expiresAt) {
        throw new Error('Invoice expired')
      }

      if (utxo.satoshis < invoice.amount + 500) {
        throw new Error('Insufficient funds')
      }

      const tx = new Transaction()

      // Add input
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(payerKey),
        sequence: 0xffffffff
      })

      // Add payment output to merchant
      tx.addOutput({
        satoshis: invoice.amount,
        lockingScript: Script.fromAddress(invoice.merchantAddress)
      })

      // Add change output
      const fee = 500
      const change = utxo.satoshis - invoice.amount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(payerKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Update invoice
      invoice.status = 'paid'
      invoice.paymentTxid = tx.id('hex')
      invoice.paidAt = Date.now()

      console.log('Invoice payment processed')
      console.log('Invoice ID:', invoice.id)
      console.log('Transaction ID:', tx.id('hex'))

      return { tx, invoice }
    } catch (error) {
      throw new Error(`Invoice payment failed: ${error.message}`)
    }
  }

  /**
   * Verify invoice payment on-chain
   */
  async verifyInvoicePayment(
    invoice: Invoice,
    tx: Transaction
  ): Promise<{
    verified: boolean
    message: string
  }> {
    try {
      if (!invoice.paymentTxid) {
        return {
          verified: false,
          message: 'Invoice has no payment transaction'
        }
      }

      // Check transaction ID matches
      if (tx.id('hex') !== invoice.paymentTxid) {
        return {
          verified: false,
          message: 'Transaction ID mismatch'
        }
      }

      // Check payment output
      let paymentFound = false
      for (const output of tx.outputs) {
        try {
          const outputAddress = output.lockingScript.toAddress()
          if (outputAddress === invoice.merchantAddress && output.satoshis === invoice.amount) {
            paymentFound = true
            break
          }
        } catch {
          // Skip non-address outputs
          continue
        }
      }

      if (!paymentFound) {
        return {
          verified: false,
          message: 'Payment output not found or amount incorrect'
        }
      }

      return {
        verified: true,
        message: 'Invoice payment verified'
      }
    } catch (error) {
      return {
        verified: false,
        message: `Verification failed: ${error.message}`
      }
    }
  }

  /**
   * Generate payment receipt
   */
  generateReceipt(invoice: Invoice): PaymentReceipt {
    if (invoice.status !== 'paid') {
      throw new Error('Invoice not paid')
    }

    return {
      invoiceId: invoice.id,
      amount: invoice.amount,
      merchantAddress: invoice.merchantAddress,
      paymentTxid: invoice.paymentTxid!,
      paidAt: invoice.paidAt!,
      description: invoice.description
    }
  }
}

interface Invoice {
  id: string
  merchantAddress: string
  amount: number
  description: string
  createdAt: number
  expiresAt: number
  status: 'pending' | 'paid' | 'expired'
  paymentTxid: string | null
  paidAt?: number
}

interface PaymentReceipt {
  invoiceId: string
  amount: number
  merchantAddress: string
  paymentTxid: string
  paidAt: number
  description: string
}

/**
 * Usage Example
 */
async function invoiceExample() {
  const processor = new InvoiceProcessor()
  const customerKey = PrivateKey.fromRandom()

  // Merchant generates invoice
  const invoice = processor.generateInvoice({
    merchantAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    amount: 50000,
    invoiceId: 'INV-001',
    description: 'Website hosting - Monthly'
  })

  // Customer pays invoice
  const utxo = {
    txid: 'customer-utxo...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(customerKey.toPublicKey().toHash())
  }

  const { tx, invoice: paidInvoice } = await processor.processInvoicePayment(
    invoice,
    customerKey,
    utxo
  )

  // Verify payment
  const verification = await processor.verifyInvoicePayment(paidInvoice, tx)
  console.log('Payment verified:', verification.verified)

  // Generate receipt
  const receipt = processor.generateReceipt(paidInvoice)
  console.log('Receipt:', receipt)
}
```

## Recurring Payments

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Recurring Payment Manager
 *
 * Manage recurring payment schedules
 */
class RecurringPaymentManager {
  /**
   * Create recurring payment schedule
   */
  createSchedule(params: {
    payerAddress: string
    payeeAddress: string
    amount: number
    frequency: 'daily' | 'weekly' | 'monthly'
    startDate: number
    endDate?: number
  }): PaymentSchedule {
    const schedule: PaymentSchedule = {
      id: this.generateScheduleId(),
      payerAddress: params.payerAddress,
      payeeAddress: params.payeeAddress,
      amount: params.amount,
      frequency: params.frequency,
      startDate: params.startDate,
      endDate: params.endDate,
      nextPaymentDate: params.startDate,
      status: 'active',
      payments: []
    }

    console.log('Recurring payment schedule created')
    console.log('Schedule ID:', schedule.id)
    console.log('Amount:', schedule.amount, 'satoshis')
    console.log('Frequency:', schedule.frequency)

    return schedule
  }

  /**
   * Process next scheduled payment
   */
  async processScheduledPayment(
    schedule: PaymentSchedule,
    payerKey: PrivateKey,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    schedule: PaymentSchedule
  }> {
    try {
      // Validate schedule
      if (schedule.status !== 'active') {
        throw new Error('Payment schedule not active')
      }

      const now = Date.now()

      if (now < schedule.nextPaymentDate) {
        throw new Error('Next payment not due yet')
      }

      if (schedule.endDate && now > schedule.endDate) {
        schedule.status = 'expired'
        throw new Error('Payment schedule expired')
      }

      // Create payment transaction
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(payerKey),
        sequence: 0xffffffff
      })

      // Payment output
      tx.addOutput({
        satoshis: schedule.amount,
        lockingScript: Script.fromAddress(schedule.payeeAddress)
      })

      // Change output
      const fee = 500
      const change = utxo.satoshis - schedule.amount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(payerKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Record payment
      const payment: ScheduledPayment = {
        txid: tx.id('hex'),
        amount: schedule.amount,
        date: now,
        status: 'completed'
      }

      schedule.payments.push(payment)
      schedule.nextPaymentDate = this.calculateNextPaymentDate(
        schedule.nextPaymentDate,
        schedule.frequency
      )

      console.log('Scheduled payment processed')
      console.log('Payment number:', schedule.payments.length)
      console.log('Next payment:', new Date(schedule.nextPaymentDate).toISOString())

      return { tx, schedule }
    } catch (error) {
      throw new Error(`Scheduled payment failed: ${error.message}`)
    }
  }

  /**
   * Cancel payment schedule
   */
  cancelSchedule(schedule: PaymentSchedule): PaymentSchedule {
    schedule.status = 'cancelled'
    console.log('Payment schedule cancelled')
    console.log('Schedule ID:', schedule.id)
    console.log('Total payments made:', schedule.payments.length)
    return schedule
  }

  /**
   * Get payment history
   */
  getPaymentHistory(schedule: PaymentSchedule): {
    totalPayments: number
    totalAmount: number
    payments: ScheduledPayment[]
  } {
    const totalPayments = schedule.payments.length
    const totalAmount = schedule.payments.reduce((sum, p) => sum + p.amount, 0)

    return {
      totalPayments,
      totalAmount,
      payments: schedule.payments
    }
  }

  /**
   * Calculate next payment date
   */
  private calculateNextPaymentDate(
    currentDate: number,
    frequency: 'daily' | 'weekly' | 'monthly'
  ): number {
    const date = new Date(currentDate)

    switch (frequency) {
      case 'daily':
        date.setDate(date.getDate() + 1)
        break
      case 'weekly':
        date.setDate(date.getDate() + 7)
        break
      case 'monthly':
        date.setMonth(date.getMonth() + 1)
        break
    }

    return date.getTime()
  }

  /**
   * Generate unique schedule ID
   */
  private generateScheduleId(): string {
    return `SCHED-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface PaymentSchedule {
  id: string
  payerAddress: string
  payeeAddress: string
  amount: number
  frequency: 'daily' | 'weekly' | 'monthly'
  startDate: number
  endDate?: number
  nextPaymentDate: number
  status: 'active' | 'cancelled' | 'expired'
  payments: ScheduledPayment[]
}

interface ScheduledPayment {
  txid: string
  amount: number
  date: number
  status: 'completed' | 'failed'
}

/**
 * Usage Example
 */
async function recurringPaymentExample() {
  const manager = new RecurringPaymentManager()
  const payerKey = PrivateKey.fromRandom()

  // Create monthly subscription
  const schedule = manager.createSchedule({
    payerAddress: payerKey.toPublicKey().toAddress(),
    payeeAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    amount: 50000,
    frequency: 'monthly',
    startDate: Date.now(),
    endDate: Date.now() + 365 * 24 * 60 * 60 * 1000 // 1 year
  })

  // Process payment
  const utxo = {
    txid: 'payer-utxo...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(payerKey.toPublicKey().toHash())
  }

  const { tx, schedule: updatedSchedule } = await manager.processScheduledPayment(
    schedule,
    payerKey,
    utxo
  )

  // Get payment history
  const history = manager.getPaymentHistory(updatedSchedule)
  console.log('Payment history:', history)
}
```

## Subscription Management

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Subscription Manager
 *
 * Manage customer subscriptions with various tiers
 */
class SubscriptionManager {
  private plans: Map<string, SubscriptionPlan> = new Map()

  /**
   * Create subscription plan
   */
  createPlan(plan: SubscriptionPlan): void {
    this.plans.set(plan.id, plan)

    console.log('Subscription plan created')
    console.log('Plan ID:', plan.id)
    console.log('Name:', plan.name)
    console.log('Price:', plan.price, 'satoshis /', plan.billingCycle)
  }

  /**
   * Subscribe customer to plan
   */
  async subscribe(params: {
    customerId: string
    customerKey: PrivateKey
    planId: string
    merchantAddress: string
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<{
    subscription: Subscription
    tx: Transaction
  }> {
    try {
      const plan = this.plans.get(params.planId)
      if (!plan) {
        throw new Error('Plan not found')
      }

      // Process initial payment
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.customerKey),
        sequence: 0xffffffff
      })

      tx.addOutput({
        satoshis: plan.price,
        lockingScript: Script.fromAddress(params.merchantAddress)
      })

      const fee = 500
      const change = params.utxo.satoshis - plan.price - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.customerKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Create subscription
      const subscription: Subscription = {
        id: this.generateSubscriptionId(),
        customerId: params.customerId,
        planId: params.planId,
        status: 'active',
        startDate: Date.now(),
        currentPeriodStart: Date.now(),
        currentPeriodEnd: this.calculatePeriodEnd(Date.now(), plan.billingCycle),
        payments: [
          {
            txid: tx.id('hex'),
            amount: plan.price,
            date: Date.now(),
            periodStart: Date.now(),
            periodEnd: this.calculatePeriodEnd(Date.now(), plan.billingCycle)
          }
        ]
      }

      console.log('Subscription created')
      console.log('Subscription ID:', subscription.id)
      console.log('Plan:', plan.name)
      console.log('Next billing:', new Date(subscription.currentPeriodEnd).toISOString())

      return { subscription, tx }
    } catch (error) {
      throw new Error(`Subscription failed: ${error.message}`)
    }
  }

  /**
   * Renew subscription
   */
  async renewSubscription(
    subscription: Subscription,
    customerKey: PrivateKey,
    merchantAddress: string,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    subscription: Subscription
    tx: Transaction
  }> {
    try {
      if (subscription.status !== 'active') {
        throw new Error('Subscription not active')
      }

      const plan = this.plans.get(subscription.planId)
      if (!plan) {
        throw new Error('Plan not found')
      }

      // Process renewal payment
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(customerKey),
        sequence: 0xffffffff
      })

      tx.addOutput({
        satoshis: plan.price,
        lockingScript: Script.fromAddress(merchantAddress)
      })

      const fee = 500
      const change = utxo.satoshis - plan.price - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(customerKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Update subscription
      const newPeriodStart = subscription.currentPeriodEnd
      const newPeriodEnd = this.calculatePeriodEnd(newPeriodStart, plan.billingCycle)

      subscription.currentPeriodStart = newPeriodStart
      subscription.currentPeriodEnd = newPeriodEnd
      subscription.payments.push({
        txid: tx.id('hex'),
        amount: plan.price,
        date: Date.now(),
        periodStart: newPeriodStart,
        periodEnd: newPeriodEnd
      })

      console.log('Subscription renewed')
      console.log('Next billing:', new Date(newPeriodEnd).toISOString())

      return { subscription, tx }
    } catch (error) {
      throw new Error(`Renewal failed: ${error.message}`)
    }
  }

  /**
   * Cancel subscription
   */
  cancelSubscription(subscription: Subscription): Subscription {
    subscription.status = 'cancelled'
    subscription.cancelledAt = Date.now()

    console.log('Subscription cancelled')
    console.log('Subscription ID:', subscription.id)
    console.log('Active until:', new Date(subscription.currentPeriodEnd).toISOString())

    return subscription
  }

  /**
   * Check if subscription is active
   */
  isSubscriptionActive(subscription: Subscription): boolean {
    if (subscription.status !== 'active') {
      return false
    }

    return Date.now() < subscription.currentPeriodEnd
  }

  /**
   * Calculate period end date
   */
  private calculatePeriodEnd(
    startDate: number,
    billingCycle: 'monthly' | 'yearly'
  ): number {
    const date = new Date(startDate)

    if (billingCycle === 'monthly') {
      date.setMonth(date.getMonth() + 1)
    } else {
      date.setFullYear(date.getFullYear() + 1)
    }

    return date.getTime()
  }

  /**
   * Generate subscription ID
   */
  private generateSubscriptionId(): string {
    return `SUB-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface SubscriptionPlan {
  id: string
  name: string
  description: string
  price: number // satoshis
  billingCycle: 'monthly' | 'yearly'
  features: string[]
}

interface Subscription {
  id: string
  customerId: string
  planId: string
  status: 'active' | 'cancelled' | 'expired'
  startDate: number
  currentPeriodStart: number
  currentPeriodEnd: number
  cancelledAt?: number
  payments: SubscriptionPayment[]
}

interface SubscriptionPayment {
  txid: string
  amount: number
  date: number
  periodStart: number
  periodEnd: number
}

/**
 * Usage Example
 */
async function subscriptionExample() {
  const manager = new SubscriptionManager()
  const customerKey = PrivateKey.fromRandom()

  // Create plans
  manager.createPlan({
    id: 'basic',
    name: 'Basic Plan',
    description: 'Basic features',
    price: 50000,
    billingCycle: 'monthly',
    features: ['Feature 1', 'Feature 2']
  })

  // Customer subscribes
  const utxo = {
    txid: 'customer-utxo...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(customerKey.toPublicKey().toHash())
  }

  const { subscription } = await manager.subscribe({
    customerId: 'customer-123',
    customerKey,
    planId: 'basic',
    merchantAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    utxo
  })

  // Check if active
  const isActive = manager.isSubscriptionActive(subscription)
  console.log('Subscription active:', isActive)
}
```

## Related Examples

- [Payment Distribution](../payment-distribution/README.md)
- [Transaction Building](../transaction-building/README.md)
- [E-commerce Integration](../ecommerce-integration/README.md)
- [Content Paywall](../content-paywall/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [P2PKH](../../sdk-components/p2pkh/README.md) - Standard payments
- [ARC](../../sdk-components/arc/README.md) - Transaction broadcasting
- [UTXO Management](../../sdk-components/utxo-management/README.md) - UTXO handling

**Learning Paths:**
- [Payment Processing](../../learning-paths/intermediate/payment-processing/README.md)
