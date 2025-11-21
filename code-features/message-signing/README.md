# Message Signing

Complete examples for signing and verifying messages with Bitcoin private keys.

## Overview

This code feature demonstrates how to sign arbitrary messages with Bitcoin private keys and verify those signatures. Message signing is essential for proving ownership of a private key without revealing it, enabling authentication, proving identity, and verifying data integrity.

**Related SDK Components:**
- [Signatures](../../sdk-components/signatures/README.md)
- [Private Keys](../../sdk-components/private-keys/README.md)
- [Public Keys](../../sdk-components/public-keys/README.md)
- [ECDSA](../../sdk-components/ecdsa/README.md)

## Basic Message Signing

```typescript
import { PrivateKey, PublicKey, Signature, Hash } from '@bsv/sdk'

/**
 * Basic Message Signing
 *
 * Sign and verify messages using Bitcoin keys
 */
class BasicMessageSigning {
  /**
   * Sign a message with a private key
   */
  signMessage(privateKey: PrivateKey, message: string): {
    signature: string
    publicKey: string
    message: string
  } {
    // Hash the message
    const messageHash = Hash.sha256(Buffer.from(message, 'utf8'))

    // Sign the hash
    const signature = privateKey.sign(messageHash)

    return {
      signature: signature.toDER().toString('hex'),
      publicKey: privateKey.toPublicKey().toString(),
      message: message
    }
  }

  /**
   * Verify a message signature
   */
  verifyMessage(
    publicKey: string,
    message: string,
    signatureHex: string
  ): boolean {
    try {
      // Reconstruct the message hash
      const messageHash = Hash.sha256(Buffer.from(message, 'utf8'))

      // Parse signature and public key
      const signature = Signature.fromDER(Buffer.from(signatureHex, 'hex'))
      const pubKey = PublicKey.fromString(publicKey)

      // Verify signature
      return pubKey.verify(messageHash, signature)
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }

  /**
   * Sign message and return compact format
   */
  signMessageCompact(privateKey: PrivateKey, message: string): string {
    const messageHash = Hash.sha256(Buffer.from(message, 'utf8'))
    const signature = privateKey.sign(messageHash)

    return signature.toCompact().toString('base64')
  }

  /**
   * Verify compact signature
   */
  verifyMessageCompact(
    publicKey: string,
    message: string,
    signatureBase64: string
  ): boolean {
    try {
      const messageHash = Hash.sha256(Buffer.from(message, 'utf8'))
      const signature = Signature.fromCompact(Buffer.from(signatureBase64, 'base64'))
      const pubKey = PublicKey.fromString(publicKey)

      return pubKey.verify(messageHash, signature)
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }
}

/**
 * Usage Example
 */
async function example() {
  const signer = new BasicMessageSigning()
  const privateKey = PrivateKey.fromRandom()

  // Sign a message
  const signed = signer.signMessage(
    privateKey,
    'Hello, Bitcoin!'
  )

  console.log('Signature:', signed.signature)
  console.log('Public Key:', signed.publicKey)

  // Verify the message
  const isValid = signer.verifyMessage(
    signed.publicKey,
    signed.message,
    signed.signature
  )

  console.log('Signature valid:', isValid)
}
```

## Bitcoin Signed Message Format

```typescript
import { PrivateKey, PublicKey, Signature, Hash } from '@bsv/sdk'

/**
 * Bitcoin Signed Message Format
 *
 * Implements the standard "Bitcoin Signed Message" format used by wallets
 */
class BitcoinSignedMessage {
  private readonly MAGIC_BYTES = '\x18Bitcoin Signed Message:\n'

  /**
   * Create Bitcoin message hash (with magic bytes)
   */
  private createMessageHash(message: string): number[] {
    const messageBuffer = Buffer.from(message, 'utf8')

    // Create the formatted message
    const prefix = Buffer.from(this.MAGIC_BYTES, 'utf8')
    const messageLength = this.encodeVarint(messageBuffer.length)

    // Concatenate: magic bytes + message length + message
    const fullMessage = Buffer.concat([
      prefix,
      messageLength,
      messageBuffer
    ])

    // Double SHA256
    return Hash.sha256(Hash.sha256(fullMessage))
  }

  /**
   * Encode number as varint
   */
  private encodeVarint(n: number): Buffer {
    if (n < 0xfd) {
      return Buffer.from([n])
    } else if (n <= 0xffff) {
      const buf = Buffer.allocUnsafe(3)
      buf.writeUInt8(0xfd, 0)
      buf.writeUInt16LE(n, 1)
      return buf
    } else if (n <= 0xffffffff) {
      const buf = Buffer.allocUnsafe(5)
      buf.writeUInt8(0xfe, 0)
      buf.writeUInt32LE(n, 1)
      return buf
    } else {
      const buf = Buffer.allocUnsafe(9)
      buf.writeUInt8(0xff, 0)
      buf.writeBigUInt64LE(BigInt(n), 1)
      return buf
    }
  }

  /**
   * Sign message using Bitcoin Signed Message format
   */
  signMessage(privateKey: PrivateKey, message: string): {
    message: string
    signature: string
    address: string
  } {
    const messageHash = this.createMessageHash(message)
    const signature = privateKey.sign(messageHash)

    return {
      message: message,
      signature: signature.toCompact().toString('base64'),
      address: privateKey.toAddress()
    }
  }

  /**
   * Verify Bitcoin Signed Message
   */
  verifyMessage(
    address: string,
    message: string,
    signatureBase64: string
  ): boolean {
    try {
      const messageHash = this.createMessageHash(message)
      const signature = Signature.fromCompact(
        Buffer.from(signatureBase64, 'base64')
      )

      // Recover public key from signature
      const publicKey = signature.RecoverPublicKey(messageHash)

      // Check if recovered address matches
      return publicKey.toAddress() === address
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }

  /**
   * Recover public key from signed message
   */
  recoverPublicKey(message: string, signatureBase64: string): PublicKey | null {
    try {
      const messageHash = this.createMessageHash(message)
      const signature = Signature.fromCompact(
        Buffer.from(signatureBase64, 'base64')
      )

      return signature.RecoverPublicKey(messageHash)
    } catch (error) {
      console.error('Recovery failed:', error)
      return null
    }
  }
}

/**
 * Usage Example
 */
async function bitcoinMessageExample() {
  const bsm = new BitcoinSignedMessage()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  // Sign a message
  const signed = bsm.signMessage(
    privateKey,
    'I own this Bitcoin address'
  )

  console.log('Message:', signed.message)
  console.log('Signature:', signed.signature)
  console.log('Address:', signed.address)

  // Verify the signature
  const isValid = bsm.verifyMessage(
    signed.address,
    signed.message,
    signed.signature
  )

  console.log('Signature valid:', isValid)

  // Recover the public key
  const recoveredPubKey = bsm.recoverPublicKey(
    signed.message,
    signed.signature
  )

  console.log('Recovered address:', recoveredPubKey?.toAddress())
}
```

## Advanced Message Signing Patterns

```typescript
import { PrivateKey, PublicKey, Signature, Hash } from '@bsv/sdk'

/**
 * Advanced Message Signing
 *
 * Complex signing patterns and use cases
 */
class AdvancedMessageSigning {
  /**
   * Sign structured data (JSON)
   */
  signStructuredData(
    privateKey: PrivateKey,
    data: object
  ): {
    data: object
    signature: string
    publicKey: string
    timestamp: number
  } {
    const timestamp = Date.now()

    // Create canonical representation
    const canonical = JSON.stringify({
      ...data,
      timestamp
    }, null, 0) // No whitespace for consistency

    // Hash and sign
    const hash = Hash.sha256(Buffer.from(canonical, 'utf8'))
    const signature = privateKey.sign(hash)

    return {
      data,
      signature: signature.toDER().toString('hex'),
      publicKey: privateKey.toPublicKey().toString(),
      timestamp
    }
  }

  /**
   * Verify structured data signature
   */
  verifyStructuredData(
    publicKey: string,
    data: object,
    signatureHex: string,
    timestamp: number
  ): boolean {
    try {
      // Recreate canonical representation
      const canonical = JSON.stringify({
        ...data,
        timestamp
      }, null, 0)

      // Verify signature
      const hash = Hash.sha256(Buffer.from(canonical, 'utf8'))
      const signature = Signature.fromDER(Buffer.from(signatureHex, 'hex'))
      const pubKey = PublicKey.fromString(publicKey)

      return pubKey.verify(hash, signature)
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }

  /**
   * Multi-message signing (sign multiple messages at once)
   */
  signMultipleMessages(
    privateKey: PrivateKey,
    messages: string[]
  ): Array<{
    message: string
    signature: string
  }> {
    return messages.map(message => {
      const hash = Hash.sha256(Buffer.from(message, 'utf8'))
      const signature = privateKey.sign(hash)

      return {
        message,
        signature: signature.toDER().toString('hex')
      }
    })
  }

  /**
   * Verify multiple message signatures
   */
  verifyMultipleMessages(
    publicKey: string,
    signedMessages: Array<{ message: string; signature: string }>
  ): boolean {
    try {
      const pubKey = PublicKey.fromString(publicKey)

      return signedMessages.every(({ message, signature }) => {
        const hash = Hash.sha256(Buffer.from(message, 'utf8'))
        const sig = Signature.fromDER(Buffer.from(signature, 'hex'))
        return pubKey.verify(hash, sig)
      })
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }

  /**
   * Sign message with expiration
   */
  signMessageWithExpiry(
    privateKey: PrivateKey,
    message: string,
    expirySeconds: number
  ): {
    message: string
    signature: string
    publicKey: string
    expiresAt: number
  } {
    const expiresAt = Date.now() + (expirySeconds * 1000)

    // Include expiry in signed data
    const dataToSign = JSON.stringify({
      message,
      expiresAt
    })

    const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
    const signature = privateKey.sign(hash)

    return {
      message,
      signature: signature.toDER().toString('hex'),
      publicKey: privateKey.toPublicKey().toString(),
      expiresAt
    }
  }

  /**
   * Verify message with expiration check
   */
  verifyMessageWithExpiry(
    publicKey: string,
    message: string,
    signatureHex: string,
    expiresAt: number
  ): { valid: boolean; expired: boolean } {
    // Check expiration
    const expired = Date.now() > expiresAt

    if (expired) {
      return { valid: false, expired: true }
    }

    try {
      // Recreate signed data
      const dataToSign = JSON.stringify({
        message,
        expiresAt
      })

      const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
      const signature = Signature.fromDER(Buffer.from(signatureHex, 'hex'))
      const pubKey = PublicKey.fromString(publicKey)

      const valid = pubKey.verify(hash, signature)

      return { valid, expired: false }
    } catch (error) {
      console.error('Verification failed:', error)
      return { valid: false, expired: false }
    }
  }

  /**
   * Sign file hash
   */
  signFileHash(privateKey: PrivateKey, fileHash: string): {
    fileHash: string
    signature: string
    publicKey: string
    signedAt: number
  } {
    const signedAt = Date.now()

    // Combine file hash with timestamp
    const dataToSign = JSON.stringify({
      fileHash,
      signedAt
    })

    const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
    const signature = privateKey.sign(hash)

    return {
      fileHash,
      signature: signature.toDER().toString('hex'),
      publicKey: privateKey.toPublicKey().toString(),
      signedAt
    }
  }

  /**
   * Verify file hash signature
   */
  verifyFileHash(
    publicKey: string,
    fileHash: string,
    signatureHex: string,
    signedAt: number
  ): boolean {
    try {
      const dataToSign = JSON.stringify({
        fileHash,
        signedAt
      })

      const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
      const signature = Signature.fromDER(Buffer.from(signatureHex, 'hex'))
      const pubKey = PublicKey.fromString(publicKey)

      return pubKey.verify(hash, signature)
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }
}
```

## Authentication and Proof of Ownership

```typescript
import { PrivateKey, PublicKey, Signature, Hash } from '@bsv/sdk'

/**
 * Authentication Patterns
 *
 * Use message signing for authentication and proof of ownership
 */
class AuthenticationPatterns {
  /**
   * Create authentication challenge
   */
  createChallenge(): {
    challenge: string
    timestamp: number
    expiresAt: number
  } {
    const challenge = Hash.sha256(
      Buffer.from(Math.random().toString() + Date.now().toString())
    ).toString('hex')

    const timestamp = Date.now()
    const expiresAt = timestamp + (5 * 60 * 1000) // 5 minutes

    return {
      challenge,
      timestamp,
      expiresAt
    }
  }

  /**
   * Sign authentication challenge
   */
  signChallenge(
    privateKey: PrivateKey,
    challenge: string,
    timestamp: number
  ): {
    challenge: string
    signature: string
    publicKey: string
    address: string
  } {
    // Sign challenge with timestamp
    const dataToSign = JSON.stringify({
      challenge,
      timestamp
    })

    const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
    const signature = privateKey.sign(hash)

    return {
      challenge,
      signature: signature.toDER().toString('hex'),
      publicKey: privateKey.toPublicKey().toString(),
      address: privateKey.toAddress()
    }
  }

  /**
   * Verify authentication challenge
   */
  verifyChallenge(
    publicKey: string,
    challenge: string,
    timestamp: number,
    signatureHex: string,
    expiresAt: number
  ): { valid: boolean; expired: boolean; message: string } {
    // Check expiration
    if (Date.now() > expiresAt) {
      return {
        valid: false,
        expired: true,
        message: 'Challenge expired'
      }
    }

    try {
      // Recreate signed data
      const dataToSign = JSON.stringify({
        challenge,
        timestamp
      })

      const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
      const signature = Signature.fromDER(Buffer.from(signatureHex, 'hex'))
      const pubKey = PublicKey.fromString(publicKey)

      const valid = pubKey.verify(hash, signature)

      return {
        valid,
        expired: false,
        message: valid ? 'Authentication successful' : 'Invalid signature'
      }
    } catch (error) {
      return {
        valid: false,
        expired: false,
        message: `Verification error: ${error.message}`
      }
    }
  }

  /**
   * Prove ownership of address
   */
  proveOwnership(
    privateKey: PrivateKey,
    address: string,
    nonce: string
  ): {
    address: string
    proof: string
    nonce: string
  } | null {
    // Verify the address matches the private key
    if (privateKey.toAddress() !== address) {
      return null
    }

    // Sign the address with nonce
    const dataToSign = JSON.stringify({
      address,
      nonce,
      message: 'I own this address'
    })

    const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
    const signature = privateKey.sign(hash)

    return {
      address,
      proof: signature.toDER().toString('hex'),
      nonce
    }
  }

  /**
   * Verify ownership proof
   */
  verifyOwnership(
    address: string,
    proof: string,
    nonce: string
  ): boolean {
    try {
      // Recreate signed data
      const dataToSign = JSON.stringify({
        address,
        nonce,
        message: 'I own this address'
      })

      const hash = Hash.sha256(Buffer.from(dataToSign, 'utf8'))
      const signature = Signature.fromDER(Buffer.from(proof, 'hex'))

      // Recover public key from signature
      const publicKey = signature.RecoverPublicKey(hash)

      // Verify address matches
      return publicKey.toAddress() === address
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }

  /**
   * Complete authentication flow
   */
  async authenticateUser(
    privateKey: PrivateKey
  ): Promise<{
    success: boolean
    message: string
    session?: {
      address: string
      authenticatedAt: number
    }
  }> {
    // Step 1: Server creates challenge
    const { challenge, timestamp, expiresAt } = this.createChallenge()

    // Step 2: Client signs challenge
    const signed = this.signChallenge(privateKey, challenge, timestamp)

    // Step 3: Server verifies signature
    const verification = this.verifyChallenge(
      signed.publicKey,
      signed.challenge,
      timestamp,
      signed.signature,
      expiresAt
    )

    if (verification.valid) {
      return {
        success: true,
        message: 'Authentication successful',
        session: {
          address: signed.address,
          authenticatedAt: Date.now()
        }
      }
    }

    return {
      success: false,
      message: verification.message
    }
  }
}

/**
 * Usage Example
 */
async function authenticationExample() {
  const auth = new AuthenticationPatterns()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  // Authenticate user
  const result = await auth.authenticateUser(privateKey)

  console.log('Authentication:', result)

  if (result.success && result.session) {
    console.log('User address:', result.session.address)
    console.log('Authenticated at:', new Date(result.session.authenticatedAt))
  }
}
```

## Related Examples

- [Transaction Signing](../transaction-signing/README.md)
- [P2PKH Template](../p2pkh-template/README.md)

## See Also

**SDK Components:**
- [Signatures](../../sdk-components/signatures/README.md) - Digital signature creation and verification
- [Private Keys](../../sdk-components/private-keys/README.md) - Private key management
- [Public Keys](../../sdk-components/public-keys/README.md) - Public key operations
- [ECDSA](../../sdk-components/ecdsa/README.md) - Elliptic curve cryptography

**Learning Paths:**
- [Digital Signatures](../../learning-paths/intermediate/digital-signatures/README.md)
