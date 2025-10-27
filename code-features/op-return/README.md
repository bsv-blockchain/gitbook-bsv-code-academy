# OP_RETURN Data Embedding

Complete examples for embedding arbitrary data on the BSV blockchain using OP_RETURN outputs.

## Overview

OP_RETURN is a Bitcoin script opcode that marks transaction outputs as provably unspendable, making it ideal for storing arbitrary data on the blockchain. This feature is widely used for timestamping, document verification, application protocols, and on-chain data storage. BSV's unbounded block size makes it practical to store substantial amounts of data using OP_RETURN.

**Related SDK Components:**
- [Script](../../sdk-components/script/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Transaction Output](../../sdk-components/transaction-output/README.md)

## Simple OP_RETURN Data Storage

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP } from '@bsv/sdk'

/**
 * Simple OP_RETURN Data Storage
 *
 * Store arbitrary data in OP_RETURN outputs
 */
class OpReturnDataStore {
  /**
   * Create an OP_RETURN script with data
   */
  createOpReturnScript(data: Buffer | Buffer[]): Script {
    const script = new Script()

    // Add OP_FALSE (OP_0) and OP_RETURN
    script.writeOpCode(OP.OP_FALSE)
    script.writeOpCode(OP.OP_RETURN)

    // Add data chunks
    const dataArray = Array.isArray(data) ? data : [data]
    for (const chunk of dataArray) {
      script.writeBin(chunk)
    }

    return script
  }

  /**
   * Store text data on blockchain
   */
  async storeText(
    senderKey: PrivateKey,
    text: string,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      // Add input
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Add OP_RETURN output with text data
      const textBuffer = Buffer.from(text, 'utf8')
      tx.addOutput({
        satoshis: 0, // OP_RETURN outputs are provably unspendable
        lockingScript: this.createOpReturnScript(textBuffer)
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

      console.log('Text stored on blockchain')
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Data size:', textBuffer.length, 'bytes')

      return tx
    } catch (error) {
      throw new Error(`Failed to store text: ${error.message}`)
    }
  }

  /**
   * Store multiple data fields
   */
  async storeMultipleFields(
    senderKey: PrivateKey,
    fields: string[],
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Convert fields to buffers
      const dataBuffers = fields.map(field => Buffer.from(field, 'utf8'))

      // Add OP_RETURN output
      tx.addOutput({
        satoshis: 0,
        lockingScript: this.createOpReturnScript(dataBuffers)
      })

      // Add change
      const fee = 500
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Multiple fields stored')
      console.log('Fields:', fields.length)
      console.log('Transaction ID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`Failed to store fields: ${error.message}`)
    }
  }

  /**
   * Parse OP_RETURN data from script
   */
  parseOpReturnData(script: Script): Buffer[] {
    const chunks = script.chunks
    const data: Buffer[] = []

    // Skip OP_FALSE (index 0) and OP_RETURN (index 1)
    for (let i = 2; i < chunks.length; i++) {
      if (chunks[i].buf) {
        data.push(chunks[i].buf)
      }
    }

    return data
  }

  /**
   * Extract OP_RETURN data from transaction
   */
  extractData(tx: Transaction): Buffer[] | null {
    for (const output of tx.outputs) {
      const chunks = output.lockingScript.chunks

      // Check if this is an OP_RETURN output
      if (chunks.length >= 2 && chunks[1].opCodeNum === OP.OP_RETURN) {
        return this.parseOpReturnData(output.lockingScript)
      }
    }

    return null
  }
}

/**
 * Usage Example
 */
async function simpleOpReturnExample() {
  const dataStore = new OpReturnDataStore()
  const privateKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx-id...',
    vout: 0,
    satoshis: 50000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Store simple text
  const tx1 = await dataStore.storeText(
    privateKey,
    'Hello, Bitcoin SV!',
    utxo
  )

  // Store multiple fields
  const tx2 = await dataStore.storeMultipleFields(
    privateKey,
    ['app_name', 'my_app', 'version', '1.0.0'],
    utxo
  )

  // Extract data
  const extractedData = dataStore.extractData(tx1)
  if (extractedData) {
    console.log('Extracted text:', extractedData[0].toString('utf8'))
  }
}
```

## Document Timestamping

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP, Hash } from '@bsv/sdk'

/**
 * Document Timestamping Service
 *
 * Creates immutable timestamps for documents using OP_RETURN
 */
class DocumentTimestamper {
  private readonly PROTOCOL_ID = 'TIMESTAMP'

  /**
   * Create timestamp proof for document
   */
  async timestampDocument(
    senderKey: PrivateKey,
    documentHash: string,
    documentName: string,
    metadata: Record<string, string>,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    timestamp: number
    proof: TimestampProof
  }> {
    try {
      const tx = new Transaction()
      const timestamp = Math.floor(Date.now() / 1000)

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Prepare timestamp data
      const dataFields = [
        Buffer.from(this.PROTOCOL_ID, 'utf8'),
        Buffer.from(documentName, 'utf8'),
        Buffer.from(documentHash, 'hex'),
        Buffer.from(timestamp.toString(), 'utf8'),
        Buffer.from(JSON.stringify(metadata), 'utf8')
      ]

      // Create OP_RETURN output
      const script = new Script()
      script.writeOpCode(OP.OP_FALSE)
      script.writeOpCode(OP.OP_RETURN)

      for (const field of dataFields) {
        script.writeBin(field)
      }

      tx.addOutput({
        satoshis: 0,
        lockingScript: script
      })

      // Change output
      const fee = 500
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      const proof: TimestampProof = {
        txid: tx.id('hex'),
        documentHash,
        documentName,
        timestamp,
        metadata,
        blockchainNetwork: 'BSV'
      }

      console.log('Document timestamped')
      console.log('Transaction ID:', proof.txid)
      console.log('Timestamp:', new Date(timestamp * 1000).toISOString())

      return { tx, timestamp, proof }
    } catch (error) {
      throw new Error(`Timestamping failed: ${error.message}`)
    }
  }

  /**
   * Timestamp multiple documents in one transaction
   */
  async timestampBatch(
    senderKey: PrivateKey,
    documents: Array<{
      hash: string
      name: string
      metadata?: Record<string, string>
    }>,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    proofs: TimestampProof[]
  }> {
    try {
      const tx = new Transaction()
      const timestamp = Math.floor(Date.now() / 1000)

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      const proofs: TimestampProof[] = []

      // Create OP_RETURN output for each document
      for (const doc of documents) {
        const dataFields = [
          Buffer.from(this.PROTOCOL_ID, 'utf8'),
          Buffer.from(doc.name, 'utf8'),
          Buffer.from(doc.hash, 'hex'),
          Buffer.from(timestamp.toString(), 'utf8')
        ]

        if (doc.metadata) {
          dataFields.push(Buffer.from(JSON.stringify(doc.metadata), 'utf8'))
        }

        const script = new Script()
        script.writeOpCode(OP.OP_FALSE)
        script.writeOpCode(OP.OP_RETURN)

        for (const field of dataFields) {
          script.writeBin(field)
        }

        tx.addOutput({
          satoshis: 0,
          lockingScript: script
        })

        proofs.push({
          txid: '', // Will be set after signing
          documentHash: doc.hash,
          documentName: doc.name,
          timestamp,
          metadata: doc.metadata || {},
          blockchainNetwork: 'BSV'
        })
      }

      // Change output
      const fee = 500 + (documents.length * 100)
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Update proofs with transaction ID
      const txid = tx.id('hex')
      proofs.forEach(proof => proof.txid = txid)

      console.log('Batch timestamp completed')
      console.log('Documents:', documents.length)
      console.log('Transaction ID:', txid)

      return { tx, proofs }
    } catch (error) {
      throw new Error(`Batch timestamping failed: ${error.message}`)
    }
  }

  /**
   * Verify timestamp proof
   */
  async verifyTimestamp(
    proof: TimestampProof,
    documentContent: Buffer
  ): Promise<{
    valid: boolean
    message: string
  }> {
    try {
      // Calculate document hash
      const calculatedHash = Hash.sha256(documentContent).toString('hex')

      if (calculatedHash !== proof.documentHash) {
        return {
          valid: false,
          message: 'Document hash does not match proof'
        }
      }

      // In production, verify the transaction exists on chain
      // and extract the OP_RETURN data to confirm it matches the proof

      return {
        valid: true,
        message: `Document was timestamped on ${new Date(proof.timestamp * 1000).toISOString()}`
      }
    } catch (error) {
      return {
        valid: false,
        message: `Verification failed: ${error.message}`
      }
    }
  }
}

interface TimestampProof {
  txid: string
  documentHash: string
  documentName: string
  timestamp: number
  metadata: Record<string, string>
  blockchainNetwork: string
}

/**
 * Usage Example
 */
async function timestampExample() {
  const timestamper = new DocumentTimestamper()
  const privateKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Timestamp a document
  const documentContent = Buffer.from('Important contract document...')
  const documentHash = Hash.sha256(documentContent).toString('hex')

  const { proof } = await timestamper.timestampDocument(
    privateKey,
    documentHash,
    'contract.pdf',
    {
      author: 'Alice',
      version: '1.0'
    },
    utxo
  )

  // Verify timestamp
  const verification = await timestamper.verifyTimestamp(proof, documentContent)
  console.log('Verification:', verification)
}
```

## JSON Data Storage

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP } from '@bsv/sdk'

/**
 * JSON Data Storage on Blockchain
 *
 * Store structured JSON data using OP_RETURN
 */
class JsonDataStore {
  private readonly MAX_PUSH_SIZE = 520 // Bitcoin script push limit

  /**
   * Store JSON object on blockchain
   */
  async storeJSON(
    senderKey: PrivateKey,
    data: object,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const jsonString = JSON.stringify(data)
      const jsonBuffer = Buffer.from(jsonString, 'utf8')

      console.log('Storing JSON data')
      console.log('Size:', jsonBuffer.length, 'bytes')

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Split data if larger than push limit
      const chunks = this.splitIntoChunks(jsonBuffer)

      // Add OP_RETURN output with JSON data
      const script = new Script()
      script.writeOpCode(OP.OP_FALSE)
      script.writeOpCode(OP.OP_RETURN)

      for (const chunk of chunks) {
        script.writeBin(chunk)
      }

      tx.addOutput({
        satoshis: 0,
        lockingScript: script
      })

      // Change output
      const fee = 500 + Math.ceil(jsonBuffer.length / 100)
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('JSON stored on blockchain')
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Chunks:', chunks.length)

      return tx
    } catch (error) {
      throw new Error(`Failed to store JSON: ${error.message}`)
    }
  }

  /**
   * Store application protocol message
   */
  async storeProtocolMessage(
    senderKey: PrivateKey,
    protocol: string,
    action: string,
    payload: object,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Protocol format: [protocol_id, action, json_payload]
      const payloadJson = JSON.stringify(payload)
      const dataFields = [
        Buffer.from(protocol, 'utf8'),
        Buffer.from(action, 'utf8'),
        ...this.splitIntoChunks(Buffer.from(payloadJson, 'utf8'))
      ]

      const script = new Script()
      script.writeOpCode(OP.OP_FALSE)
      script.writeOpCode(OP.OP_RETURN)

      for (const field of dataFields) {
        script.writeBin(field)
      }

      tx.addOutput({
        satoshis: 0,
        lockingScript: script
      })

      // Change output
      const fee = 500
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Protocol message stored')
      console.log('Protocol:', protocol)
      console.log('Action:', action)
      console.log('Transaction ID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`Failed to store protocol message: ${error.message}`)
    }
  }

  /**
   * Extract JSON data from transaction
   */
  extractJSON(tx: Transaction): object | null {
    try {
      for (const output of tx.outputs) {
        const chunks = output.lockingScript.chunks

        if (chunks.length >= 2 && chunks[1].opCodeNum === OP.OP_RETURN) {
          // Collect all data chunks
          const dataChunks: Buffer[] = []
          for (let i = 2; i < chunks.length; i++) {
            if (chunks[i].buf) {
              dataChunks.push(chunks[i].buf)
            }
          }

          // Combine chunks and parse JSON
          const combinedData = Buffer.concat(dataChunks)
          const jsonString = combinedData.toString('utf8')
          return JSON.parse(jsonString)
        }
      }

      return null
    } catch (error) {
      console.error('Failed to extract JSON:', error.message)
      return null
    }
  }

  /**
   * Split buffer into chunks that fit in script pushes
   */
  private splitIntoChunks(buffer: Buffer): Buffer[] {
    const chunks: Buffer[] = []
    let offset = 0

    while (offset < buffer.length) {
      const chunkSize = Math.min(this.MAX_PUSH_SIZE, buffer.length - offset)
      chunks.push(buffer.subarray(offset, offset + chunkSize))
      offset += chunkSize
    }

    return chunks
  }
}

/**
 * Usage Example
 */
async function jsonStorageExample() {
  const store = new JsonDataStore()
  const privateKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 50000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Store JSON object
  const data = {
    type: 'user_profile',
    username: 'alice',
    created_at: new Date().toISOString(),
    preferences: {
      theme: 'dark',
      notifications: true
    }
  }

  const tx = await store.storeJSON(privateKey, data, utxo)

  // Extract JSON
  const extracted = store.extractJSON(tx)
  console.log('Extracted data:', extracted)

  // Store protocol message
  const msgTx = await store.storeProtocolMessage(
    privateKey,
    'MY_APP',
    'CREATE_POST',
    {
      title: 'Hello World',
      content: 'My first post on BSV',
      tags: ['blockchain', 'bsv']
    },
    utxo
  )
}
```

## File Metadata Registry

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP, Hash } from '@bsv/sdk'

/**
 * File Metadata Registry
 *
 * Register file metadata and hashes on blockchain
 */
class FileMetadataRegistry {
  private readonly PROTOCOL = 'FILE_REGISTRY'

  /**
   * Register file metadata on blockchain
   */
  async registerFile(
    senderKey: PrivateKey,
    file: {
      name: string
      hash: string
      size: number
      mimeType: string
      url?: string
      tags?: string[]
    },
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    tx: Transaction
    fileId: string
  }> {
    try {
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      const metadata = {
        protocol: this.PROTOCOL,
        name: file.name,
        hash: file.hash,
        size: file.size,
        mimeType: file.mimeType,
        url: file.url || '',
        tags: file.tags || [],
        registeredAt: Math.floor(Date.now() / 1000)
      }

      const metadataJson = JSON.stringify(metadata)
      const metadataBuffer = Buffer.from(metadataJson, 'utf8')

      // Create OP_RETURN with metadata
      const script = new Script()
      script.writeOpCode(OP.OP_FALSE)
      script.writeOpCode(OP.OP_RETURN)
      script.writeBin(Buffer.from(this.PROTOCOL, 'utf8'))
      script.writeBin(metadataBuffer)

      tx.addOutput({
        satoshis: 0,
        lockingScript: script
      })

      // Change output
      const fee = 500
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      const fileId = tx.id('hex')

      console.log('File registered on blockchain')
      console.log('File ID:', fileId)
      console.log('Name:', file.name)
      console.log('Hash:', file.hash)

      return { tx, fileId }
    } catch (error) {
      throw new Error(`File registration failed: ${error.message}`)
    }
  }

  /**
   * Register file with proof of ownership
   */
  async registerWithOwnership(
    ownerKey: PrivateKey,
    file: {
      name: string
      hash: string
      size: number
      mimeType: string
    },
    license: string,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(ownerKey),
        sequence: 0xffffffff
      })

      // Create ownership proof
      const ownerAddress = ownerKey.toPublicKey().toAddress()
      const ownershipData = {
        protocol: this.PROTOCOL,
        action: 'REGISTER_WITH_OWNERSHIP',
        owner: ownerAddress,
        file: file,
        license: license,
        timestamp: Math.floor(Date.now() / 1000)
      }

      const dataJson = JSON.stringify(ownershipData)

      const script = new Script()
      script.writeOpCode(OP.OP_FALSE)
      script.writeOpCode(OP.OP_RETURN)
      script.writeBin(Buffer.from(this.PROTOCOL, 'utf8'))
      script.writeBin(Buffer.from('OWNERSHIP', 'utf8'))
      script.writeBin(Buffer.from(dataJson, 'utf8'))

      tx.addOutput({
        satoshis: 0,
        lockingScript: script
      })

      // Change output
      const fee = 500
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(ownerKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('File registered with ownership proof')
      console.log('Owner:', ownerAddress)
      console.log('Transaction ID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`Ownership registration failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function fileRegistryExample() {
  const registry = new FileMetadataRegistry()
  const privateKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 50000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Register file
  const fileContent = Buffer.from('File content here...')
  const fileHash = Hash.sha256(fileContent).toString('hex')

  const { fileId } = await registry.registerFile(
    privateKey,
    {
      name: 'document.pdf',
      hash: fileHash,
      size: fileContent.length,
      mimeType: 'application/pdf',
      url: 'https://example.com/document.pdf',
      tags: ['contract', 'legal']
    },
    utxo
  )

  console.log('File registered with ID:', fileId)

  // Register with ownership
  await registry.registerWithOwnership(
    privateKey,
    {
      name: 'artwork.jpg',
      hash: fileHash,
      size: 1024000,
      mimeType: 'image/jpeg'
    },
    'CC BY-SA 4.0',
    utxo
  )
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Custom Scripts](../custom-scripts/README.md)
- [Standard Transactions](../standard-transactions/README.md)
- [Content Paywall](../content-paywall/README.md)

## See Also

**SDK Components:**
- [Script](../../sdk-components/script/README.md) - Script creation and manipulation
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [Transaction Output](../../sdk-components/transaction-output/README.md) - Output management

**Learning Paths:**
- [Bitcoin Script](../../learning-paths/intermediate/bitcoin-script/README.md)
- [Data Storage](../../learning-paths/advanced/blockchain-data-storage/README.md)
- [Protocol Development](../../learning-paths/advanced/protocol-development/README.md)
