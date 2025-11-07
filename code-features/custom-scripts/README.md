# Custom Bitcoin Scripts

Complete examples for creating custom Bitcoin scripts beyond standard templates.

## Overview

This code feature demonstrates how to create custom Bitcoin scripts for advanced use cases. While P2PKH is the most common script type, Bitcoin Script supports a wide range of functionality including time locks, hash locks, custom validation logic, and complex spending conditions.

**Related SDK Components:**
- [Script](../../sdk-components/script/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [OP Codes](../../sdk-components/op-codes/README.md)

## Hash Lock Script

```typescript
import { Transaction, PrivateKey, Script, OP, Hash } from '@bsv/sdk'

/**
 * Hash Lock Script
 *
 * Requires knowledge of a secret (preimage) to spend
 */
class HashLockScript {
  /**
   * Create hash lock locking script
   * Requires: <preimage> <signature> <pubkey>
   */
  createLockingScript(secretHash: Buffer, publicKeyHash: number[]): Script {
    const script = new Script()

    // Verify preimage matches hash
    script.writeOpCode(OP.OP_HASH256)
    script.writeBin(secretHash)
    script.writeOpCode(OP.OP_EQUALVERIFY)

    // Standard P2PKH verification
    script.writeOpCode(OP.OP_DUP)
    script.writeOpCode(OP.OP_HASH160)
    script.writeBin(Buffer.from(publicKeyHash))
    script.writeOpCode(OP.OP_EQUALVERIFY)
    script.writeOpCode(OP.OP_CHECKSIG)

    return script
  }

  /**
   * Create unlocking script for hash lock
   */
  createUnlockingScript(
    preimage: Buffer,
    signature: Buffer,
    publicKey: Buffer
  ): Script {
    const script = new Script()

    script.writeBin(signature)
    script.writeBin(publicKey)
    script.writeBin(preimage)

    return script
  }

  /**
   * Lock funds with hash lock
   */
  async lockFunds(
    senderKey: PrivateKey,
    secret: string,
    recipientPubKeyHash: number[],
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<{ tx: Transaction; secretHash: Buffer }> {
    // Create hash of secret
    const secretHash = Hash.sha256(Hash.sha256(Buffer.from(secret, 'utf8')))

    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    // Add hash-locked output
    tx.addOutput({
      satoshis: amount,
      lockingScript: this.createLockingScript(secretHash, recipientPubKeyHash)
    })

    // Add change
    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()

    return { tx, secretHash }
  }

  /**
   * Spend hash-locked funds
   */
  async spendHashLock(
    recipientKey: PrivateKey,
    secret: string,
    hashLockUtxo: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: hashLockUtxo.txid,
      sourceOutputIndex: hashLockUtxo.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Create signature
    const preimage = tx.inputs[0].getPreimage(hashLockUtxo.lockingScript)
    const signature = recipientKey.sign(preimage)

    // Create unlocking script
    const unlockingScript = this.createUnlockingScript(
      Buffer.from(secret, 'utf8'),
      signature.toDER(),
      recipientKey.toPublicKey().encode()
    )

    tx.inputs[0].unlockingScript = unlockingScript

    return tx
  }
}

/**
 * Usage Example
 */
async function hashLockExample() {
  const hashLock = new HashLockScript()

  const sender = PrivateKey.fromRandom()
  const recipient = PrivateKey.fromRandom()
  const secret = 'my-secret-phrase'

  // Lock funds
  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000
  }

  const { tx: lockTx, secretHash } = await hashLock.lockFunds(
    sender,
    secret,
    recipient.toPublicKey().toHash(),
    50000,
    utxo
  )

  console.log('Hash locked:', lockTx.id('hex'))
  console.log('Secret hash:', secretHash.toString('hex'))

  // Spend by revealing secret
  const hashLockUtxo = {
    txid: lockTx.id('hex'),
    vout: 0,
    satoshis: 50000,
    lockingScript: hashLock.createLockingScript(
      secretHash,
      recipient.toPublicKey().toHash()
    )
  }

  const spendTx = await hashLock.spendHashLock(
    recipient,
    secret,
    hashLockUtxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    49000
  )

  console.log('Hash lock spent:', spendTx.id('hex'))
}
```

## Time Lock Script (CLTV)

```typescript
import { Transaction, PrivateKey, Script, OP, P2PKH } from '@bsv/sdk'

/**
 * Time Lock Script using CheckLockTimeVerify (CLTV)
 *
 * Funds cannot be spent until a specific time/block height
 */
class TimeLockScript {
  /**
   * Create time-locked locking script
   */
  createLockingScript(
    lockTime: number,
    publicKeyHash: number[]
  ): Script {
    const script = new Script()

    // Push lock time and verify
    script.writeBin(this.numberToBuffer(lockTime))
    script.writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
    script.writeOpCode(OP.OP_DROP)

    // Standard P2PKH
    script.writeOpCode(OP.OP_DUP)
    script.writeOpCode(OP.OP_HASH160)
    script.writeBin(Buffer.from(publicKeyHash))
    script.writeOpCode(OP.OP_EQUALVERIFY)
    script.writeOpCode(OP.OP_CHECKSIG)

    return script
  }

  /**
   * Convert number to buffer for script
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])

    const isNegative = num < 0
    const absNum = Math.abs(num)

    const hex = absNum.toString(16)
    const paddedHex = hex.length % 2 ? '0' + hex : hex
    const buf = Buffer.from(paddedHex, 'hex').reverse()

    if (isNegative) {
      buf[buf.length - 1] |= 0x80
    }

    return buf
  }

  /**
   * Lock funds until specific time
   */
  async lockUntilTime(
    senderKey: PrivateKey,
    unlockTime: number,
    recipientPubKeyHash: number[],
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    // Add time-locked output
    tx.addOutput({
      satoshis: amount,
      lockingScript: this.createLockingScript(unlockTime, recipientPubKeyHash)
    })

    // Add change
    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Spend time-locked funds
   */
  async spendTimeLock(
    recipientKey: PrivateKey,
    lockTime: number,
    timeLockUtxo: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Set transaction lock time
    tx.lockTime = lockTime

    tx.addInput({
      sourceTXID: timeLockUtxo.txid,
      sourceOutputIndex: timeLockUtxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(recipientKey),
      sequence: 0xfffffffe // Must be less than 0xffffffff for CLTV
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    await tx.sign()
    return tx
  }

  /**
   * Create time lock for specific block height
   */
  async lockUntilBlockHeight(
    senderKey: PrivateKey,
    blockHeight: number,
    recipientPubKeyHash: number[],
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    return this.lockUntilTime(
      senderKey,
      blockHeight,
      recipientPubKeyHash,
      amount,
      utxo
    )
  }

  /**
   * Create time lock for Unix timestamp
   */
  async lockUntilTimestamp(
    senderKey: PrivateKey,
    timestamp: number,
    recipientPubKeyHash: number[],
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    // CLTV uses Unix timestamp if >= 500000000
    if (timestamp < 500000000) {
      throw new Error('Timestamp must be >= 500000000 for CLTV')
    }

    return this.lockUntilTime(
      senderKey,
      timestamp,
      recipientPubKeyHash,
      amount,
      utxo
    )
  }
}

/**
 * Usage Example
 */
async function timeLockExample() {
  const timeLock = new TimeLockScript()

  const sender = PrivateKey.fromRandom()
  const recipient = PrivateKey.fromRandom()

  // Lock until specific timestamp (1 hour from now)
  const unlockTime = Math.floor(Date.now() / 1000) + 3600

  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000
  }

  const lockTx = await timeLock.lockUntilTimestamp(
    sender,
    unlockTime,
    recipient.toPublicKey().toHash(),
    50000,
    utxo
  )

  console.log('Time locked until:', new Date(unlockTime * 1000))
  console.log('Transaction:', lockTx.id('hex'))

  // Later, after unlock time...
  const timeLockUtxo = {
    txid: lockTx.id('hex'),
    vout: 0,
    satoshis: 50000,
    lockingScript: timeLock.createLockingScript(
      unlockTime,
      recipient.toPublicKey().toHash()
    )
  }

  const spendTx = await timeLock.spendTimeLock(
    recipient,
    unlockTime,
    timeLockUtxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    49000
  )

  console.log('Time lock spent:', spendTx.id('hex'))
}
```

## Puzzle Script

```typescript
import { Transaction, PrivateKey, Script, OP, Hash } from '@bsv/sdk'

/**
 * Puzzle Scripts
 *
 * Anyone who knows the solution can spend
 */
class PuzzleScript {
  /**
   * Simple math puzzle: requires providing two numbers that add to target
   */
  createAdditionPuzzle(targetSum: number): Script {
    const script = new Script()

    // Stack: <num1> <num2>
    script.writeOpCode(OP.OP_ADD)  // Stack: <sum>
    script.writeBin(this.numberToBuffer(targetSum))  // Stack: <sum> <target>
    script.writeOpCode(OP.OP_EQUAL)  // Stack: <true/false>

    return script
  }

  /**
   * Hash puzzle: requires providing data that hashes to specific value
   */
  createHashPuzzle(targetHash: Buffer): Script {
    const script = new Script()

    // Stack: <data>
    script.writeOpCode(OP.OP_HASH256)  // Stack: <hash>
    script.writeBin(targetHash)  // Stack: <hash> <target>
    script.writeOpCode(OP.OP_EQUAL)  // Stack: <true/false>

    return script
  }

  /**
   * String puzzle: requires providing exact string
   */
  createStringPuzzle(targetString: string): Script {
    const script = new Script()

    // Stack: <string>
    script.writeBin(Buffer.from(targetString, 'utf8'))  // Stack: <string> <target>
    script.writeOpCode(OP.OP_EQUAL)  // Stack: <true/false>

    return script
  }

  /**
   * Lock funds to puzzle
   */
  async lockToPuzzle(
    senderKey: PrivateKey,
    puzzleScript: Script,
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: puzzleScript
    })

    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Solve addition puzzle
   */
  solveAdditionPuzzle(num1: number, num2: number): Script {
    const script = new Script()

    script.writeBin(this.numberToBuffer(num1))
    script.writeBin(this.numberToBuffer(num2))

    return script
  }

  /**
   * Solve hash puzzle
   */
  solveHashPuzzle(data: string): Script {
    const script = new Script()
    script.writeBin(Buffer.from(data, 'utf8'))
    return script
  }

  /**
   * Solve string puzzle
   */
  solveStringPuzzle(answer: string): Script {
    const script = new Script()
    script.writeBin(Buffer.from(answer, 'utf8'))
    return script
  }

  /**
   * Spend from puzzle
   */
  async spendPuzzle(
    solutionScript: Script,
    puzzleUtxo: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: puzzleUtxo.txid,
      sourceOutputIndex: puzzleUtxo.vout,
      unlockingScript: solutionScript,
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    return tx
  }

  /**
   * Helper to convert number to buffer
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])

    const isNegative = num < 0
    const absNum = Math.abs(num)

    const hex = absNum.toString(16)
    const paddedHex = hex.length % 2 ? '0' + hex : hex
    const buf = Buffer.from(paddedHex, 'hex').reverse()

    if (isNegative) {
      buf[buf.length - 1] |= 0x80
    }

    return buf
  }
}

/**
 * Usage Example
 */
async function puzzleExample() {
  const puzzle = new PuzzleScript()
  const sender = PrivateKey.fromRandom()

  // Create addition puzzle: find two numbers that add to 42
  const puzzleScript = puzzle.createAdditionPuzzle(42)

  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000
  }

  // Lock funds to puzzle
  const lockTx = await puzzle.lockToPuzzle(
    sender,
    puzzleScript,
    50000,
    utxo
  )

  console.log('Puzzle locked:', lockTx.id('hex'))
  console.log('Solve: find two numbers that add to 42')

  // Anyone can solve and claim
  const solution = puzzle.solveAdditionPuzzle(20, 22)

  const puzzleUtxo = {
    txid: lockTx.id('hex'),
    vout: 0,
    satoshis: 50000,
    lockingScript: puzzleScript
  }

  const spendTx = await puzzle.spendPuzzle(
    solution,
    puzzleUtxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    49000
  )

  console.log('Puzzle solved:', spendTx.id('hex'))
}
```

## OP_RETURN Data Script

```typescript
import { Transaction, PrivateKey, Script, OP, P2PKH } from '@bsv/sdk'

/**
 * OP_RETURN Scripts
 *
 * Store arbitrary data on the blockchain
 */
class OpReturnScript {
  /**
   * Create OP_RETURN output with data
   */
  createOpReturnScript(data: Buffer | Buffer[]): Script {
    const script = new Script()

    script.writeOpCode(OP.OP_FALSE)  // or OP.OP_RETURN
    script.writeOpCode(OP.OP_RETURN)

    // Add data chunks
    if (Array.isArray(data)) {
      for (const chunk of data) {
        script.writeBin(chunk)
      }
    } else {
      script.writeBin(data)
    }

    return script
  }

  /**
   * Create transaction with OP_RETURN output
   */
  async createDataTransaction(
    senderKey: PrivateKey,
    data: string | Buffer | Buffer[],
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    // Convert string to buffer if needed
    let dataBuffers: Buffer[]
    if (typeof data === 'string') {
      dataBuffers = [Buffer.from(data, 'utf8')]
    } else if (Array.isArray(data)) {
      dataBuffers = data
    } else {
      dataBuffers = [data]
    }

    // Add OP_RETURN output
    tx.addOutput({
      satoshis: 0,  // OP_RETURN outputs are unspendable
      lockingScript: this.createOpReturnScript(dataBuffers)
    })

    // Add change output
    const fee = 500
    const change = utxo.satoshis - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Store JSON data
   */
  async storeJSON(
    senderKey: PrivateKey,
    jsonData: object,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const jsonString = JSON.stringify(jsonData)
    return this.createDataTransaction(senderKey, jsonString, utxo)
  }

  /**
   * Store file hash
   */
  async storeFileHash(
    senderKey: PrivateKey,
    fileHash: string,
    fileName: string,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const data = [
      Buffer.from('FILE_HASH', 'utf8'),
      Buffer.from(fileName, 'utf8'),
      Buffer.from(fileHash, 'hex')
    ]

    return this.createDataTransaction(senderKey, data, utxo)
  }

  /**
   * Parse OP_RETURN data from script
   */
  parseOpReturnData(script: Script): Buffer[] {
    const chunks = script.chunks
    const data: Buffer[] = []

    // Skip OP_FALSE and OP_RETURN
    for (let i = 2; i < chunks.length; i++) {
      if (chunks[i].buf) {
        data.push(chunks[i].buf)
      }
    }

    return data
  }

  /**
   * Extract OP_RETURN from transaction
   */
  extractOpReturnData(tx: Transaction): Buffer[] | null {
    for (const output of tx.outputs) {
      const script = output.lockingScript
      const chunks = script.chunks

      // Check if this is an OP_RETURN output
      if (chunks.length >= 2 && chunks[1].opCodeNum === OP.OP_RETURN) {
        return this.parseOpReturnData(script)
      }
    }

    return null
  }
}

/**
 * Usage Example
 */
async function opReturnExample() {
  const opReturn = new OpReturnScript()
  const sender = PrivateKey.fromRandom()

  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 10000
  }

  // Store simple text
  const textTx = await opReturn.createDataTransaction(
    sender,
    'Hello, Bitcoin!',
    utxo
  )

  console.log('Text stored:', textTx.id('hex'))

  // Store JSON
  const jsonTx = await opReturn.storeJSON(
    sender,
    { type: 'document', version: 1, data: 'example' },
    utxo
  )

  console.log('JSON stored:', jsonTx.id('hex'))

  // Extract data
  const extractedData = opReturn.extractOpReturnData(textTx)
  if (extractedData) {
    console.log('Extracted:', extractedData[0].toString('utf8'))
  }
}
```

## Related Examples

- [P2PKH Template](../p2pkh-template/README.md)
- [Multi-Signature](../multi-signature/README.md)
- [Transaction Building](../transaction-building/README.md)
- [OP_RETURN](../op-return/README.md)

## See Also

**SDK Components:**
- [Script](../../sdk-components/script/README.md) - Script creation and manipulation
- [OP Codes](../../sdk-components/op-codes/README.md) - Bitcoin Script opcodes
- [Script Templates](../../sdk-components/script-templates/README.md) - Template system
- [Transaction](../../sdk-components/transaction/README.md) - Transaction construction

**Learning Paths:**
- [Script Templates](../../learning-paths/intermediate/script-templates/README.md)
- [Bitcoin Script](../../learning-paths/advanced/bitcoin-script/README.md)
