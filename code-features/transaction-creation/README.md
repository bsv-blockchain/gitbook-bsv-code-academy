# Transaction Creation

## Overview

This example demonstrates how to create a basic BSV transaction from scratch, including adding inputs, outputs, and calculating fees.

## Prerequisites

- BSV TypeScript SDK installed
- A private key with available UTXOs
- Basic understanding of Bitcoin transactions

## Code Example

```typescript
import { Transaction, P2PKH, PrivateKey, SatoshisPerKilobyte } from '@bsv/sdk'

async function createBasicTransaction() {
  // 1. Set up your private key
  const privateKey = PrivateKey.fromWif('your-private-key-wif')
  const sourceAddress = privateKey.toPublicKey().toAddress()

  // 2. Create a new transaction
  const tx = new Transaction()

  // 3. Add input from a UTXO you own
  const utxo = {
    txid: 'previous-transaction-id',
    vout: 0,
    satoshis: 10000,
    script: new P2PKH().lock(sourceAddress)
  }

  await tx.addInput({
    sourceTransaction: utxo.txid,
    sourceOutputIndex: utxo.vout,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey),
    sequence: 0xffffffff
  })

  // 4. Add output (payment to recipient)
  const recipientAddress = 'recipient-bsv-address'
  const paymentAmount = 5000 // satoshis

  tx.addOutput({
    satoshis: paymentAmount,
    lockingScript: new P2PKH().lock(recipientAddress)
  })

  // 5. Calculate and add change output
  const feeModel = new SatoshisPerKilobyte(50) // 50 sats/KB
  const estimatedFee = await tx.getFee(feeModel)
  const changeAmount = utxo.satoshis - paymentAmount - estimatedFee

  if (changeAmount > 0) {
    tx.addOutput({
      satoshis: changeAmount,
      lockingScript: new P2PKH().lock(sourceAddress)
    })
  }

  // 6. Sign the transaction
  await tx.sign()

  // 7. Serialize for broadcasting
  const rawTx = tx.toHex()
  console.log('Transaction hex:', rawTx)
  console.log('Transaction ID:', tx.id('hex'))

  return tx
}

// Run the example
createBasicTransaction()
  .then(tx => console.log('Transaction created successfully'))
  .catch(err => console.error('Error creating transaction:', err))
```

## Explanation

### Step-by-Step Breakdown

1. **Private Key Setup**: Import or generate the private key that will sign the transaction
2. **Transaction Creation**: Initialize a new empty transaction
3. **Add Input**: Reference a UTXO you control, specifying how to unlock it
4. **Add Output**: Create payment to recipient with locking script
5. **Change Calculation**: Calculate fees and create change output back to yourself
6. **Signing**: Sign the transaction with your private key
7. **Serialization**: Convert to hex format for broadcasting

## Key Concepts

### Inputs
Inputs reference previous transaction outputs (UTXOs) that you're spending. You must provide:
- Transaction ID containing the UTXO
- Output index within that transaction
- Unlocking script template (how to unlock the funds)

### Outputs
Outputs create new UTXOs that can be spent later. Each output has:
- Amount in satoshis
- Locking script (conditions to spend)

### Fees
Fees = Total Input Amount - Total Output Amount. The fee should be appropriate for the transaction size to ensure timely mining.

## Common Variations

### Multiple Inputs
```typescript
// Add multiple UTXOs as inputs
for (const utxo of utxos) {
  await tx.addInput({
    sourceTransaction: utxo.txid,
    sourceOutputIndex: utxo.vout,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  })
}
```

### Multiple Recipients
```typescript
// Pay multiple addresses
const recipients = [
  { address: 'addr1', amount: 1000 },
  { address: 'addr2', amount: 2000 }
]

for (const recipient of recipients) {
  tx.addOutput({
    satoshis: recipient.amount,
    lockingScript: new P2PKH().lock(recipient.address)
  })
}
```

## Related Components

- [Transaction SDK Component](../../sdk-components/transaction/README.md)
- [P2PKH Template](../../sdk-components/p2pkh/README.md)
- [Private Keys](../../sdk-components/private-keys/README.md)

## Related Code Features

- [Fee Estimation](../fee-estimation/README.md)
- [Sign Transaction](../sign-transaction/README.md)
- [Broadcast with ARC](../broadcast-arc/README.md)

## Learning Path References

- Beginner: [Your First Transaction](../../learning-paths/beginner/first-transaction/README.md)
- Intermediate: [Transaction Building](../../learning-paths/intermediate/transaction-building/README.md)

## Error Handling

```typescript
try {
  const tx = await createBasicTransaction()
} catch (error) {
  if (error.message.includes('insufficient funds')) {
    console.error('Not enough satoshis in UTXO')
  } else if (error.message.includes('signing failed')) {
    console.error('Invalid private key or script')
  } else {
    console.error('Unexpected error:', error)
  }
}
```

## Best Practices

1. Always verify UTXO ownership before spending
2. Use appropriate fee rates for network conditions
3. Validate output amounts are positive
4. Keep change outputs above dust threshold
5. Test with small amounts first
