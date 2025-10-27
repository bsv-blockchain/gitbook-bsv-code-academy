# Payment Distribution

Complete examples for distributing payments to multiple recipients including revenue sharing, affiliate payouts, and royalty distribution.

## Overview

Payment distribution allows a single transaction to split funds among multiple recipients according to predefined rules. This is essential for revenue sharing, affiliate programs, royalty payments, and collaborative applications. BSV's low fees make micro-splits economically viable.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Transaction Output](../../sdk-components/transaction-output/README.md)
- [P2PKH](../../sdk-components/p2pkh/README.md)
- [UTXO Management](../../sdk-components/utxo-management/README.md)

## Simple Payment Splitting

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Payment Splitter
 *
 * Split a payment among multiple recipients
 */
class PaymentSplitter {
  /**
   * Split payment by percentages
   */
  async splitByPercentage(params: {
    senderKey: PrivateKey
    totalAmount: number
    splits: Array<{
      address: string
      percentage: number // 0-100
    }>
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<{
    tx: Transaction
    distributions: Array<{
      address: string
      amount: number
      percentage: number
    }>
  }> {
    try {
      // Validate percentages
      const totalPercentage = params.splits.reduce((sum, s) => sum + s.percentage, 0)
      if (Math.abs(totalPercentage - 100) > 0.01) {
        throw new Error(`Percentages must sum to 100, got ${totalPercentage}`)
      }

      const tx = new Transaction()

      // Add input
      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.senderKey),
        sequence: 0xffffffff
      })

      const distributions: Array<{
        address: string
        amount: number
        percentage: number
      }> = []

      // Calculate and add recipient outputs
      let totalDistributed = 0

      for (let i = 0; i < params.splits.length; i++) {
        const split = params.splits[i]
        let amount: number

        // Last recipient gets remainder to handle rounding
        if (i === params.splits.length - 1) {
          amount = params.totalAmount - totalDistributed
        } else {
          amount = Math.floor((params.totalAmount * split.percentage) / 100)
          totalDistributed += amount
        }

        // Check dust threshold
        if (amount < 546) {
          throw new Error(`Split amount ${amount} below dust threshold for ${split.address}`)
        }

        tx.addOutput({
          satoshis: amount,
          lockingScript: Script.fromAddress(split.address)
        })

        distributions.push({
          address: split.address,
          amount,
          percentage: split.percentage
        })
      }

      // Add change output
      const fee = 500 + (params.splits.length * 50)
      const change = params.utxo.satoshis - params.totalAmount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Payment split completed')
      console.log('Recipients:', params.splits.length)
      console.log('Total distributed:', params.totalAmount)
      console.log('Transaction ID:', tx.id('hex'))

      return { tx, distributions }
    } catch (error) {
      throw new Error(`Payment split failed: ${error.message}`)
    }
  }

  /**
   * Split payment by fixed amounts
   */
  async splitByAmount(params: {
    senderKey: PrivateKey
    splits: Array<{
      address: string
      amount: number
    }>
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<Transaction> {
    try {
      const totalAmount = params.splits.reduce((sum, s) => sum + s.amount, 0)
      const fee = 500 + (params.splits.length * 50)

      if (params.utxo.satoshis < totalAmount + fee) {
        throw new Error('Insufficient funds')
      }

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.senderKey),
        sequence: 0xffffffff
      })

      // Add recipient outputs
      for (const split of params.splits) {
        if (split.amount < 546) {
          throw new Error(`Amount ${split.amount} below dust threshold for ${split.address}`)
        }

        tx.addOutput({
          satoshis: split.amount,
          lockingScript: Script.fromAddress(split.address)
        })
      }

      // Add change
      const change = params.utxo.satoshis - totalAmount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Fixed amount split completed')
      console.log('Recipients:', params.splits.length)
      console.log('Total:', totalAmount)

      return tx
    } catch (error) {
      throw new Error(`Fixed split failed: ${error.message}`)
    }
  }

  /**
   * Equal split among recipients
   */
  async splitEqually(params: {
    senderKey: PrivateKey
    totalAmount: number
    recipientAddresses: string[]
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<Transaction> {
    try {
      const amountPerRecipient = Math.floor(
        params.totalAmount / params.recipientAddresses.length
      )

      if (amountPerRecipient < 546) {
        throw new Error('Split amount below dust threshold')
      }

      const splits = params.recipientAddresses.map(address => ({
        address,
        amount: amountPerRecipient
      }))

      return await this.splitByAmount({
        senderKey: params.senderKey,
        splits,
        utxo: params.utxo
      })
    } catch (error) {
      throw new Error(`Equal split failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function paymentSplitterExample() {
  const splitter = new PaymentSplitter()
  const senderKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 200000,
    script: new P2PKH().lock(senderKey.toPublicKey().toHash())
  }

  // Split by percentage
  const { distributions } = await splitter.splitByPercentage({
    senderKey,
    totalAmount: 100000,
    splits: [
      { address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', percentage: 50 },
      { address: '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2', percentage: 30 },
      { address: '1HLoD9E4SDFFPDiYfNYnkBLQ85Y51J3Zb1', percentage: 20 }
    ],
    utxo
  })

  console.log('Distributions:', distributions)

  // Equal split
  await splitter.splitEqually({
    senderKey,
    totalAmount: 90000,
    recipientAddresses: [
      '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
      '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2',
      '1HLoD9E4SDFFPDiYfNYnkBLQ85Y51J3Zb1'
    ],
    utxo
  })
}
```

## Revenue Sharing

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Revenue Sharing Manager
 *
 * Automate revenue distribution among stakeholders
 */
class RevenueShareManager {
  private shares: Map<string, RevenueShare> = new Map()

  /**
   * Define revenue sharing agreement
   */
  defineShares(
    agreementId: string,
    stakeholders: Array<{
      name: string
      address: string
      percentage: number
    }>
  ): RevenueShare {
    const totalPercentage = stakeholders.reduce((sum, s) => sum + s.percentage, 0)

    if (Math.abs(totalPercentage - 100) > 0.01) {
      throw new Error('Stakeholder percentages must sum to 100')
    }

    const share: RevenueShare = {
      agreementId,
      stakeholders,
      totalDistributed: 0,
      distributionHistory: []
    }

    this.shares.set(agreementId, share)

    console.log('Revenue share agreement created')
    console.log('Agreement ID:', agreementId)
    console.log('Stakeholders:', stakeholders.length)

    return share
  }

  /**
   * Distribute revenue according to agreement
   */
  async distributeRevenue(
    agreementId: string,
    distributorKey: PrivateKey,
    revenue: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    distribution: Distribution
  }> {
    try {
      const share = this.shares.get(agreementId)
      if (!share) {
        throw new Error('Revenue share agreement not found')
      }

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(distributorKey),
        sequence: 0xffffffff
      })

      const distribution: Distribution = {
        date: Date.now(),
        totalRevenue: revenue,
        stakeholderPayments: []
      }

      let totalDistributed = 0

      // Calculate and create outputs for each stakeholder
      for (let i = 0; i < share.stakeholders.length; i++) {
        const stakeholder = share.stakeholders[i]
        let amount: number

        // Last stakeholder gets remainder
        if (i === share.stakeholders.length - 1) {
          amount = revenue - totalDistributed
        } else {
          amount = Math.floor((revenue * stakeholder.percentage) / 100)
          totalDistributed += amount
        }

        if (amount >= 546) {
          tx.addOutput({
            satoshis: amount,
            lockingScript: Script.fromAddress(stakeholder.address)
          })

          distribution.stakeholderPayments.push({
            stakeholder: stakeholder.name,
            address: stakeholder.address,
            amount,
            percentage: stakeholder.percentage
          })
        }
      }

      // Add change
      const fee = 500 + (share.stakeholders.length * 50)
      const change = utxo.satoshis - revenue - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(distributorKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      distribution.txid = tx.id('hex')

      // Update share records
      share.totalDistributed += revenue
      share.distributionHistory.push(distribution)

      console.log('Revenue distributed')
      console.log('Total:', revenue)
      console.log('Payments:', distribution.stakeholderPayments.length)
      console.log('Transaction ID:', tx.id('hex'))

      return { tx, distribution }
    } catch (error) {
      throw new Error(`Revenue distribution failed: ${error.message}`)
    }
  }

  /**
   * Get distribution history
   */
  getDistributionHistory(agreementId: string): Distribution[] {
    const share = this.shares.get(agreementId)
    if (!share) {
      throw new Error('Agreement not found')
    }
    return share.distributionHistory
  }

  /**
   * Calculate stakeholder earnings
   */
  getStakeholderEarnings(
    agreementId: string,
    stakeholderName: string
  ): {
    totalEarnings: number
    paymentCount: number
    averagePayment: number
  } {
    const share = this.shares.get(agreementId)
    if (!share) {
      throw new Error('Agreement not found')
    }

    let totalEarnings = 0
    let paymentCount = 0

    for (const distribution of share.distributionHistory) {
      for (const payment of distribution.stakeholderPayments) {
        if (payment.stakeholder === stakeholderName) {
          totalEarnings += payment.amount
          paymentCount++
        }
      }
    }

    return {
      totalEarnings,
      paymentCount,
      averagePayment: paymentCount > 0 ? totalEarnings / paymentCount : 0
    }
  }
}

interface RevenueShare {
  agreementId: string
  stakeholders: Array<{
    name: string
    address: string
    percentage: number
  }>
  totalDistributed: number
  distributionHistory: Distribution[]
}

interface Distribution {
  date: number
  totalRevenue: number
  txid?: string
  stakeholderPayments: Array<{
    stakeholder: string
    address: string
    amount: number
    percentage: number
  }>
}

/**
 * Usage Example
 */
async function revenueShareExample() {
  const manager = new RevenueShareManager()
  const distributorKey = PrivateKey.fromRandom()

  // Define revenue share
  manager.defineShares('AGREEMENT-001', [
    { name: 'Founder', address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', percentage: 40 },
    { name: 'Developer', address: '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2', percentage: 30 },
    { name: 'Marketing', address: '1HLoD9E4SDFFPDiYfNYnkBLQ85Y51J3Zb1', percentage: 20 },
    { name: 'Operations', address: '1Q2TWHE3GMdB6BZKafqwxXtWAWgFt5Jvm3', percentage: 10 }
  ])

  // Distribute revenue
  const utxo = {
    txid: 'revenue-utxo...',
    vout: 0,
    satoshis: 500000,
    script: new P2PKH().lock(distributorKey.toPublicKey().toHash())
  }

  const { distribution } = await manager.distributeRevenue(
    'AGREEMENT-001',
    distributorKey,
    200000,
    utxo
  )

  console.log('Distribution:', distribution)

  // Get stakeholder earnings
  const earnings = manager.getStakeholderEarnings('AGREEMENT-001', 'Founder')
  console.log('Founder earnings:', earnings)
}
```

## Affiliate Payouts

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Affiliate Payout Manager
 *
 * Manage affiliate commission payments
 */
class AffiliatePayoutManager {
  private affiliates: Map<string, Affiliate> = new Map()

  /**
   * Register affiliate
   */
  registerAffiliate(
    affiliateId: string,
    address: string,
    commissionRate: number // percentage
  ): Affiliate {
    const affiliate: Affiliate = {
      id: affiliateId,
      address,
      commissionRate,
      totalSales: 0,
      totalCommissions: 0,
      pendingCommissions: 0,
      sales: [],
      payouts: []
    }

    this.affiliates.set(affiliateId, affiliate)

    console.log('Affiliate registered')
    console.log('Affiliate ID:', affiliateId)
    console.log('Commission rate:', commissionRate + '%')

    return affiliate
  }

  /**
   * Record sale and calculate commission
   */
  recordSale(
    affiliateId: string,
    saleAmount: number,
    saleId: string
  ): {
    commission: number
    affiliate: Affiliate
  } {
    const affiliate = this.affiliates.get(affiliateId)
    if (!affiliate) {
      throw new Error('Affiliate not found')
    }

    const commission = Math.floor((saleAmount * affiliate.commissionRate) / 100)

    const sale: AffiliateSale = {
      saleId,
      amount: saleAmount,
      commission,
      date: Date.now(),
      paid: false
    }

    affiliate.sales.push(sale)
    affiliate.totalSales += saleAmount
    affiliate.pendingCommissions += commission

    console.log('Sale recorded for affiliate', affiliateId)
    console.log('Sale amount:', saleAmount)
    console.log('Commission:', commission)

    return { commission, affiliate }
  }

  /**
   * Pay pending commissions to affiliates
   */
  async payCommissions(
    payerKey: PrivateKey,
    affiliateIds: string[],
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    payouts: AffiliatePayout[]
  }> {
    try {
      // Calculate total payout
      let totalPayout = 0
      const payoutData: Array<{
        affiliateId: string
        address: string
        amount: number
      }> = []

      for (const affiliateId of affiliateIds) {
        const affiliate = this.affiliates.get(affiliateId)
        if (!affiliate) {
          throw new Error(`Affiliate ${affiliateId} not found`)
        }

        if (affiliate.pendingCommissions >= 546) {
          totalPayout += affiliate.pendingCommissions
          payoutData.push({
            affiliateId,
            address: affiliate.address,
            amount: affiliate.pendingCommissions
          })
        }
      }

      if (payoutData.length === 0) {
        throw new Error('No affiliates with sufficient pending commissions')
      }

      const fee = 500 + (payoutData.length * 50)

      if (utxo.satoshis < totalPayout + fee) {
        throw new Error('Insufficient funds for payout')
      }

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(payerKey),
        sequence: 0xffffffff
      })

      const payouts: AffiliatePayout[] = []

      // Create outputs for each affiliate
      for (const payout of payoutData) {
        tx.addOutput({
          satoshis: payout.amount,
          lockingScript: Script.fromAddress(payout.address)
        })

        const affiliate = this.affiliates.get(payout.affiliateId)!
        const affiliatePayout: AffiliatePayout = {
          affiliateId: payout.affiliateId,
          amount: payout.amount,
          date: Date.now(),
          txid: ''
        }

        payouts.push(affiliatePayout)

        // Update affiliate records
        affiliate.totalCommissions += payout.amount
        affiliate.pendingCommissions = 0

        // Mark sales as paid
        for (const sale of affiliate.sales) {
          if (!sale.paid) {
            sale.paid = true
          }
        }

        affiliate.payouts.push(affiliatePayout)
      }

      // Add change
      const change = utxo.satoshis - totalPayout - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(payerKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      const txid = tx.id('hex')

      // Update payout records with txid
      for (const payout of payouts) {
        payout.txid = txid
      }

      console.log('Affiliate commissions paid')
      console.log('Affiliates:', payoutData.length)
      console.log('Total:', totalPayout)
      console.log('Transaction ID:', txid)

      return { tx, payouts }
    } catch (error) {
      throw new Error(`Commission payout failed: ${error.message}`)
    }
  }

  /**
   * Get affiliate statistics
   */
  getAffiliateStats(affiliateId: string): {
    totalSales: number
    totalCommissions: number
    pendingCommissions: number
    salesCount: number
    payoutsCount: number
    conversionRate: number
  } {
    const affiliate = this.affiliates.get(affiliateId)
    if (!affiliate) {
      throw new Error('Affiliate not found')
    }

    return {
      totalSales: affiliate.totalSales,
      totalCommissions: affiliate.totalCommissions,
      pendingCommissions: affiliate.pendingCommissions,
      salesCount: affiliate.sales.length,
      payoutsCount: affiliate.payouts.length,
      conversionRate: affiliate.commissionRate
    }
  }
}

interface Affiliate {
  id: string
  address: string
  commissionRate: number
  totalSales: number
  totalCommissions: number
  pendingCommissions: number
  sales: AffiliateSale[]
  payouts: AffiliatePayout[]
}

interface AffiliateSale {
  saleId: string
  amount: number
  commission: number
  date: number
  paid: boolean
}

interface AffiliatePayout {
  affiliateId: string
  amount: number
  date: number
  txid: string
}

/**
 * Usage Example
 */
async function affiliatePayoutExample() {
  const manager = new AffiliatePayoutManager()
  const payerKey = PrivateKey.fromRandom()

  // Register affiliates
  manager.registerAffiliate(
    'AFF-001',
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    10 // 10% commission
  )

  manager.registerAffiliate(
    'AFF-002',
    '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2',
    15 // 15% commission
  )

  // Record sales
  manager.recordSale('AFF-001', 100000, 'SALE-001')
  manager.recordSale('AFF-001', 50000, 'SALE-002')
  manager.recordSale('AFF-002', 200000, 'SALE-003')

  // Pay commissions
  const utxo = {
    txid: 'payout-utxo...',
    vout: 0,
    satoshis: 500000,
    script: new P2PKH().lock(payerKey.toPublicKey().toHash())
  }

  const { payouts } = await manager.payCommissions(
    payerKey,
    ['AFF-001', 'AFF-002'],
    utxo
  )

  console.log('Payouts:', payouts)

  // Get affiliate stats
  const stats = manager.getAffiliateStats('AFF-001')
  console.log('Affiliate stats:', stats)
}
```

## Related Examples

- [Payment Processing](../payment-processing/README.md)
- [Transaction Building](../transaction-building/README.md)
- [Batch Operations](../batch-operations/README.md)
- [Marketplace](../marketplace/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [Transaction Output](../../sdk-components/transaction-output/README.md) - Output management
- [P2PKH](../../sdk-components/p2pkh/README.md) - Standard payments
- [UTXO Management](../../sdk-components/utxo-management/README.md) - UTXO handling

**Learning Paths:**
- [Payment Distribution](../../learning-paths/intermediate/payment-distribution/README.md)
- [Business Applications](../../learning-paths/advanced/business-applications/README.md)
- [Multi-Party Transactions](../../learning-paths/advanced/multi-party-transactions/README.md)
