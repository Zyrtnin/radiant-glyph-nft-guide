# Radiant Blockchain Glyph NFT Implementation Guide

**Complete Technical Documentation for Building NFT Applications on Radiant**

This guide provides everything you need to implement Glyph NFTs on the Radiant blockchain. It includes critical discoveries from real-world implementation that are not documented elsewhere.

---

## Table of Contents

1. [Overview](#overview)
2. [Critical Requirements (Read First!)](#critical-requirements-read-first)
3. [Prerequisites](#prerequisites)
4. [On-Chain Images: The `main` Field](#on-chain-images-the-main-field)
5. [Architecture](#architecture)
6. [CBOR Payload Format](#cbor-payload-format)
7. [Commit Transaction](#commit-transaction)
8. [Reveal Transaction](#reveal-transaction)
9. [Signing Challenge](#signing-challenge)
10. [Fee Calculations & Cost Analysis](#fee-calculations--cost-analysis)
11. [IPFS Integration](#ipfs-integration)
12. [Common Errors & Solutions](#common-errors--solutions)
13. [Complete Implementation Example](#complete-implementation-example)
14. [Testing & Verification](#testing--verification)
15. [Appendix: Quick Reference](#appendix-quick-reference)

---

## Overview

### What are Glyph NFTs?

Glyph NFTs are a protocol for creating non-fungible tokens on the Radiant blockchain. They support:

- **True NFTs** - Transferable, unique, visible in explorers and wallets
- **Singleton Ref Technology** - Uses `OP_PUSHINPUTREFSINGLETON` for uniqueness
- **CBOR Encoding** - Binary encoding for metadata (REQUIRED - see below)
- **On-Chain Images** - Embedded thumbnails for wallet display (REQUIRED - see below)
- **Container/Author Refs** - Organize NFTs into collections with provenance

### Two-Transaction Pattern

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Funding    │      │   Commit     │      │   Reveal     │
│    UTXO      │ ───► │     TX       │ ───► │     TX       │
│  (P2PKH)     │      │(nftCommit)   │      │(singleton)   │
└──────────────┘      └──────────────┘      └──────────────┘
                            │                      │
                            ▼                      ▼
                      Custom script          Singleton ref
                      validates glyph        NFT output
```

**Why Two Transactions?**

1. **Commit TX** - Creates an output with custom script that validates the glyph data
2. **Reveal TX** - Spends the commit output, embedding glyph data in scriptSig and creating the final NFT

This pattern ensures the NFT data is validated before the NFT is created.

### Container Organization: Commit vs Reveal Addresses

**IMPORTANT:** The commit address determines which container your NFTs belong to, while the reveal destination address determines who owns the NFT.

```
┌─────────────────────────────────────────────────────────┐
│ COMMIT TRANSACTION                                       │
│ - Address: Admin/Platform wallet                        │
│ - Purpose: Groups all NFTs into platform container      │
│ - Result: UTXOs and change stay with platform          │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ REVEAL TRANSACTION                                       │
│ - destAddress: Player wallet OR admin wallet            │
│ - Purpose: Determines who owns the NFT                  │
│ - Result: NFT singleton output to owner's address       │
└─────────────────────────────────────────────────────────┘
```

**Best Practice for Platform NFTs:**
- **Commit address:** Always use your platform's wallet (keeps all NFTs in one container)
- **Reveal destAddress:** Use player's wallet if available, otherwise your platform wallet

This design ensures:
1. All NFTs are organized in your platform's container (easy to browse in explorers)
2. Players own their NFTs (can transfer and manage them)
3. Clean fund management (UTXOs stay with platform wallet)

---

## Critical Requirements (Read First!)

> **These three requirements are essential. Missing any of them will result in broken or invisible NFTs.**

### 1. CBOR Encoding is MANDATORY

**Problem:** If you encode NFT metadata with JSON instead of CBOR, your NFT will appear as "Unknown NFT" in Glyph wallets with no metadata visible.

**Solution:** Always use CBOR encoding. Verify the library is loaded before minting:

```javascript
if (typeof CBOR === 'undefined') {
    throw new Error('CBOR library not loaded! NFT would be unreadable.');
}
```

### 2. On-Chain Thumbnail is MANDATORY

**Problem:** If you only include an IPFS link (`loc` field), your NFT will display as a blank card in Glyph Explorer and Glyphium wallet.

**Solution:** Embed a thumbnail image in the `main` field:

```javascript
payload.main = {
    t: 'image/jpeg',    // MIME type
    b: thumbnailBytes   // Uint8Array of image data
};
```

### 3. Correct Script Construction

**Problem:** The singleton output script must NOT include a push opcode before the 36-byte ref.

```
CORRECT: d8<ref>7576a914...
WRONG:   d824<ref>7576a914...  // The 24 breaks it
```

---

## Prerequisites

### Software Requirements

1. **Radiant Node** - Running with RPC access
   ```bash
   # Docker example
   docker run -d --name radiant-node \
     -p 7332:7332 \
     radiantblockchain/radiant:latest
   ```

2. **Node.js** - v18+ for signing scripts
   ```bash
   node --version  # Should be 18.0.0 or higher
   ```

3. **radiantjs Library** - For transaction signing
   ```bash
   npm install @radiantblockchain/radiantjs
   ```

4. **CBOR Library** - For metadata encoding

   **For Node.js/backend:**
   ```bash
   npm install cbor
   ```

   **For browser/frontend - Download and include in HTML:**
   ```bash
   curl -o js/cbor.min.js "https://raw.githubusercontent.com/paroga/cbor-js/master/cbor.js"
   ```

   **Load in HTML (BEFORE blockchain scripts):**
   ```html
   <!-- CBOR must load FIRST -->
   <script src="js/cbor.min.js"></script>
   <!-- Then blockchain code -->
   <script src="js/radiant_chain.js"></script>
   <script src="js/glyph_minter.js"></script>
   ```

   **Verify it's loaded:**
   ```javascript
   console.log(typeof CBOR);  // Must output "object", NOT "undefined"
   ```

### Knowledge Requirements

- Basic understanding of Bitcoin-style transactions
- Familiarity with hex encoding and byte manipulation
- JavaScript or PHP programming
- Understanding of public/private key cryptography

---

## On-Chain Images: The `main` Field

> **This is the most commonly missed requirement.** Without on-chain image data, your NFT will be invisible in wallets.

### Why On-Chain Images Are Required

Glyph wallets (Glyphium, Glyph Explorer) look for the `main` field to display NFT images. If this field is missing or empty:
- The NFT appears as a blank card
- No image preview is shown
- Only the NFT name (if readable) appears

IPFS links in the `loc` field are for **full-resolution backup**, not wallet display.

### The `main` Field Structure

```javascript
{
    p: [2],
    name: "My NFT",
    main: {
        t: 'image/jpeg',           // MIME type
        b: thumbnailUint8Array     // Image data as Uint8Array
    },
    loc: 'ipfs://Qm...'            // Full-res backup (optional)
}
```

### Thumbnail Size vs Cost Tradeoffs

| Format | Dimensions | Quality | Approx. Bytes | Approx. Cost |
|--------|------------|---------|---------------|--------------|
| JPEG   | 150px      | 65%     | 8-10 KB       | ~0.10 RXD    |
| JPEG   | 200px      | 75%     | 14-18 KB      | ~0.15 RXD    |
| WebP   | 200px      | 85%     | 15-17 KB      | ~0.17 RXD    |
| **WebP** | **225px** | **90%** | **20-25 KB** | **~0.22 RXD** |
| PNG    | 200px      | lossless| 70-85 KB      | ~0.85 RXD    |

**Recommendation:** Use **WebP format** at 225px max dimension with 90% quality. This provides:
- Sharp, clear images with excellent detail retention
- Best quality-to-size ratio of any format
- Good balance between quality and transaction costs (~0.22 RXD)
- Full compatibility with Glyphium wallet and Glyph Explorer

**Why WebP over JPEG?**
- WebP produces fewer compression artifacts at equivalent file sizes
- Single lossy compression step (capture as PNG, compress to WebP) vs double (JPEG capture → JPEG thumbnail)
- Supported by all modern browsers and Glyph wallets
- At 90% quality, produces noticeably sharper images than JPEG at similar file sizes

### JavaScript Thumbnail Creation

```javascript
async createThumbnail(dataUrl, maxSize = 225, quality = 0.90) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => {
            // Calculate dimensions maintaining aspect ratio
            let width = img.width;
            let height = img.height;

            if (width > height) {
                if (width > maxSize) {
                    height = Math.round((height * maxSize) / width);
                    width = maxSize;
                }
            } else {
                if (height > maxSize) {
                    width = Math.round((width * maxSize) / height);
                    height = maxSize;
                }
            }

            // Create canvas and draw scaled image
            const canvas = document.createElement('canvas');
            canvas.width = width;
            canvas.height = height;
            const ctx = canvas.getContext('2d');
            ctx.imageSmoothingEnabled = true;
            ctx.imageSmoothingQuality = 'high';
            ctx.drawImage(img, 0, 0, width, height);

            // Convert to WebP for best quality/size ratio
            canvas.toBlob((blob) => {
                const reader = new FileReader();
                reader.onload = () => {
                    const uint8Array = new Uint8Array(reader.result);
                    console.log(`Thumbnail: ${width}x${height}, ${uint8Array.length} bytes (WebP)`);
                    resolve(uint8Array);
                };
                reader.onerror = reject;
                reader.readAsArrayBuffer(blob);
            }, 'image/webp', quality);
        };
        img.onerror = reject;
        img.src = dataUrl;
    });
}
```

### Adding `main` Field to Payload

```javascript
function createGlyphPayload(photoData, metadata) {
    const payload = {
        p: [2],  // NFT protocol
        name: metadata.name,
        type: metadata.type || 'photo',
        attrs: metadata.attrs || {}
    };

    // CRITICAL: Add thumbnail for wallet display
    if (photoData.thumbnailData instanceof Uint8Array) {
        payload.main = {
            t: 'image/webp',  // WebP for best quality/size
            b: photoData.thumbnailData
        };
    }

    // Add IPFS for full resolution (optional)
    if (photoData.ipfsUrl) {
        payload.loc = photoData.ipfsUrl;
    }

    return payload;
}
```

---

## Architecture

### Transaction Components

#### Commit Transaction
```
Inputs:  [Funding UTXO(s)]
Outputs: [nftCommitScript Output, Change Output (optional)]

nftCommitScript validates:
  1. Glyph payload hash (double SHA256 of CBOR)
  2. "gly" marker presence
  3. Singleton ref in reveal transaction outputs
  4. P2PKH signature
```

#### Reveal Transaction
```
Inputs:  [Commit Output]
         ScriptSig: <signature> <pubkey> <"gly"> <CBOR payload>

Outputs: [Singleton NFT Output]
         Script: OP_PUSHINPUTREFSINGLETON <36-byte ref> OP_DROP <P2PKH>
```

### Key Data Structures

**36-Byte Ref Format:**
```
ref = reversed(commitTxid) + littleEndian(commitVout)
    = 32 bytes              + 4 bytes
```

**Example:**
```javascript
Commit TXID: 6afb402d085b2214b44853dad42499b5a02b823153e2789bb2bfb0e522693c26
Reversed:    263c6922e5b0bfb29b78e25331823ba0b59924d4da5348b414225b082d40fb6a
Vout: 0
Vout LE:     00000000
Ref:         263c6922e5b0bfb29b78e25331823ba0b59924d4da5348b414225b082d40fb6a00000000
```

---

## CBOR Payload Format

### Glyph Protocol Structure

```javascript
{
    p: [2],                           // Protocol: 2 = NFT (REQUIRED)
    name: "My NFT #12345",            // Token name (REQUIRED)
    type: "photo",                    // Type (optional)
    main: {                           // On-chain image (REQUIRED for display!)
        t: 'image/jpeg',              // MIME type
        b: thumbnailUint8Array        // Image bytes
    },
    in: [containerRefBytes],          // Container ref (optional)
    by: [authorRefBytes],             // Author ref (optional)
    attrs: {                          // Custom attributes (optional)
        score: 1000000,
        game: "Game Name",
        player: "Player Name"
    },
    loc: "ipfs://..."                 // IPFS full-res location (optional)
}
```

### Protocol Identifiers

| Value | Meaning |
|-------|---------|
| 1 | Fungible Token |
| 2 | Non-Fungible Token (NFT) |
| 3 | Data Storage |
| 4 | Decentralized Mint |
| 5 | Mutable |

### Container and Author Refs

Refs are 36-byte values that reference other NFTs:

```javascript
function hexToUint8Array(hex) {
    const bytes = new Uint8Array(hex.length / 2);
    for (let i = 0; i < hex.length; i += 2) {
        bytes[i / 2] = parseInt(hex.substr(i, 2), 16);
    }
    return bytes;
}

const payload = {
    p: [2],
    name: "My NFT",
    in: [hexToUint8Array(containerRef)],  // 72 hex chars
    by: [hexToUint8Array(authorRef)]      // 72 hex chars
};
```

### JavaScript CBOR Encoding

```javascript
function encodeGlyphData(data) {
    // Verify CBOR library is loaded
    if (typeof CBOR === 'undefined') {
        throw new Error('CBOR library not loaded! NFT would be unreadable.');
    }

    // Ensure protocol is set
    if (!data.p) {
        data.p = [2];
    }

    // Encode to CBOR
    const marker = new TextEncoder().encode('gly');
    const cborData = CBOR.encode(data);

    // CBOR.encode returns ArrayBuffer, convert to Uint8Array
    const payload = new Uint8Array(cborData);

    // Combine marker and payload
    const result = new Uint8Array(marker.length + payload.length);
    result.set(marker, 0);
    result.set(payload, marker.length);

    return result;
}
```

### Glyph Data Format

```
Glyph = "gly" marker (3 bytes) + CBOR payload
      = 676c79 + <CBOR bytes>
```

---

## Commit Transaction

### Purpose

The commit transaction creates an output with a custom script (nftCommitScript) that:
1. Validates the glyph data hash
2. Checks for "gly" marker
3. Verifies singleton ref exists in reveal outputs
4. Performs standard P2PKH signature validation

### nftCommitScript Structure

```
OP_HASH256 <32-byte-payload-hash> OP_EQUALVERIFY   // Verify CBOR hash
<3-byte "gly"> OP_EQUALVERIFY                       // Check "gly" marker
OP_INPUTINDEX OP_OUTPOINTTXHASH                    // Push input txid
OP_INPUTINDEX OP_OUTPOINTINDEX                     // Push input vout
OP_4 OP_NUM2BIN OP_CAT                             // Build 36-byte ref
OP_REFTYPE_OUTPUT OP_2 OP_NUMEQUALVERIFY           // Verify singleton in output
OP_DUP OP_HASH160 <20-byte-pubkeyhash> OP_EQUALVERIFY OP_CHECKSIG  // P2PKH
```

### Hex Breakdown

| Hex | Opcode | Description |
|-----|--------|-------------|
| `aa` | OP_HASH256 | Hash top stack item (double SHA256) |
| `20` | Push 32 bytes | Payload hash follows |
| `88` | OP_EQUALVERIFY | Check hash matches |
| `03` | Push 3 bytes | "gly" marker follows |
| `676c79` | "gly" | Literal bytes (ASCII: g=67, l=6c, y=79) |
| `c0` | OP_INPUTINDEX | Push current input index |
| `c8` | OP_OUTPOINTTXHASH | Push input's txid (BCH introspection) |
| `c9` | OP_OUTPOINTINDEX | Push input's vout (BCH introspection) |
| `54` | OP_4 | Push number 4 |
| `80` | OP_NUM2BIN | Convert vout to 4-byte LE |
| `7e` | OP_CAT | Concatenate txid+vout = 36-byte ref |
| `da` | OP_REFTYPE_OUTPUT | Check ref type in outputs |
| `52` | OP_2 | Push 2 (singleton type) |
| `9d` | OP_NUMEQUALVERIFY | Verify ref is singleton |
| `76a914...88ac` | P2PKH | Standard signature verification |

### PHP Implementation

```php
private function buildNftCommitScript($pubkeyhash, $payloadHash) {
    $OP_HASH256 = 'aa';
    $OP_EQUALVERIFY = '88';
    $OP_DUP = '76';
    $OP_HASH160 = 'a9';
    $OP_CHECKSIG = 'ac';
    $OP_INPUTINDEX = 'c0';
    $OP_OUTPOINTTXHASH = 'c8';  // BCH introspection
    $OP_OUTPOINTINDEX = 'c9';   // BCH introspection
    $OP_4 = '54';
    $OP_NUM2BIN = '80';
    $OP_CAT = '7e';
    $OP_REFTYPE_OUTPUT = 'da';
    $OP_2 = '52';
    $OP_NUMEQUALVERIFY = '9d';

    $glyphMarker = '676c79';  // "gly" in hex

    $script = $OP_HASH256;
    $script .= '20' . $payloadHash;
    $script .= $OP_EQUALVERIFY;
    $script .= '03' . $glyphMarker;
    $script .= $OP_EQUALVERIFY;
    $script .= $OP_INPUTINDEX . $OP_OUTPOINTTXHASH;
    $script .= $OP_INPUTINDEX . $OP_OUTPOINTINDEX;
    $script .= $OP_4 . $OP_NUM2BIN . $OP_CAT;
    $script .= $OP_REFTYPE_OUTPUT . $OP_2 . $OP_NUMEQUALVERIFY;
    $script .= $OP_DUP . $OP_HASH160;
    $script .= '14' . $pubkeyhash;
    $script .= $OP_EQUALVERIFY . $OP_CHECKSIG;

    return $script;
}
```

### Payload Hash Calculation

```php
// $glyphHex = "676c79" + <CBOR hex>
$cborHex = substr($glyphHex, 6);  // Skip first 6 hex chars ("gly")
$cborBinary = hex2bin($cborHex);
$hash1 = hash('sha256', $cborBinary, true);   // First SHA256 (binary)
$hash2 = hash('sha256', $hash1, false);        // Second SHA256 (hex)
$payloadHash = $hash2;  // This goes in the nftCommitScript
```

---

## Reveal Transaction

### Purpose

The reveal transaction spends the commit output and creates the final NFT with:
1. Glyph data embedded in scriptSig
2. Singleton ref output that references the commit transaction
3. NFT owned by specified pubkeyhash

### ScriptSig Structure (CRITICAL)

The scriptSig must be in this **exact order**:

```
<signature> <pubkey> <"gly" marker> <CBOR payload>
```

### Singleton Output Script

```
d8 <36-byte-ref> 75 76 a9 14 <20-byte-pubkeyhash> 88 ac
```

| Hex | Opcode | Description |
|-----|--------|-------------|
| `d8` | OP_PUSHINPUTREFSINGLETON | Consumes next 36 bytes as ref |
| (36 bytes) | ref | txid (reversed) + vout (LE) |
| `75` | OP_DROP | Drop the ref from stack |
| `76a914...88ac` | P2PKH | Standard P2PKH script |

**CRITICAL:** `d8` directly consumes the next 36 bytes. Do NOT add a push opcode:

- **Correct:** `d8<ref>7576a914...`
- **WRONG:** `d824<ref>7576a914...`

### JavaScript Implementation

```javascript
function buildSingletonScript(commitTxid, commitVout, ownerPubkeyhash) {
    const txidReversed = Buffer.from(commitTxid, 'hex').reverse().toString('hex');
    const voutLE = Buffer.alloc(4);
    voutLE.writeUInt32LE(commitVout);
    const ref = txidReversed + voutLE.toString('hex');

    // d8 + ref + 75 (OP_DROP) + 76a914 + pubkeyhash + 88ac
    const script = 'd8' + ref + '7576a914' + ownerPubkeyhash + '88ac';

    return { script, ref };
}
```

---

## Signing Challenge

### The Problem

The wallet RPC (`signrawtransactionwithwallet`) cannot sign transactions spending nftCommitScript because:
1. It's a custom script, not recognized P2PKH
2. The wallet doesn't know how to construct the correct scriptSig with glyph data

**Error:** "Unable to sign input, invalid stack size"

### The Solution: External Signing with Node.js

Use radiantjs library to sign the reveal transaction.

### Node.js Signing Script

```javascript
#!/usr/bin/env node
const { Script, Transaction, PrivateKey, crypto } = require('@radiantblockchain/radiantjs');

async function signReveal(params) {
    const { commitTxid, commitVout, wif, glyphHex, outputSats,
            commitScript, commitAmount, destPubkeyhash } = params;

    const privateKey = PrivateKey.fromWIF(wif);
    const publicKey = privateKey.toPublicKey();
    const pubkeyhash = destPubkeyhash ||
        crypto.Hash.sha256ripemd160(publicKey.toBuffer()).toString('hex');

    // Create 36-byte ref
    const txidReversed = Buffer.from(commitTxid, 'hex').reverse().toString('hex');
    const voutLE = Buffer.alloc(4);
    voutLE.writeUInt32LE(commitVout);
    const ref = txidReversed + voutLE.toString('hex');

    // Build singleton output script
    const singletonScript = 'd8' + ref + '7576a914' + pubkeyhash + '88ac';

    // Build transaction
    const tx = new Transaction();

    tx.addInput(new Transaction.Input({
        prevTxId: commitTxid,
        outputIndex: commitVout,
        script: new Script(),
        output: new Transaction.Output({
            script: Script.fromHex(commitScript),
            satoshis: commitAmount,
        }),
    }));

    tx.addOutput(new Transaction.Output({
        script: Script.fromHex(singletonScript),
        satoshis: outputSats
    }));

    // Build glyph scriptSig
    const glyMarker = Buffer.from('gly', 'utf8');
    const cborHex = glyphHex.substring(6);
    const cborBuffer = Buffer.from(cborHex, 'hex');

    // CRITICAL: Sign with FULL commit script, NOT P2PKH subscript
    tx.setInputScript(0, (txObj, output) => {
        const sigType = crypto.Signature.SIGHASH_ALL | crypto.Signature.SIGHASH_FORKID;

        const sig = Transaction.Sighash.sign(
            txObj,
            privateKey,
            sigType,
            0,
            output.script,  // MUST be full nftCommitScript
            new crypto.BN(String(commitAmount))
        );

        const sigBuffer = Buffer.concat([sig.toBuffer(), Buffer.from([sigType])]);

        // Build scriptSig: <sig> <pubkey> <"gly"> <CBOR>
        return Script.empty()
            .add(sigBuffer)
            .add(publicKey.toBuffer())
            .add(glyMarker)
            .add(cborBuffer)
            .toString();
    });

    tx.seal();

    return {
        success: true,
        signedTx: tx.toString(),
        txid: tx.id,
        ref: ref
    };
}
```

---

## Fee Calculations & Cost Analysis

### Radiant Fee Structure

**Radiant uses sat/byte** (NOT sat/kB like Bitcoin):
- Minimum relay fee: **1000 sat/byte**
- Always add 50% safety margin

### Common Fee Bug

```php
// WRONG - treats feeRate as sat/kB
$feeSats = ($txSize * $feeRate) / 1000;

// CORRECT - feeRate is already sat/byte
$feeSats = $txSize * $feeRate;
```

### Real-World Cost Examples

Based on verified mainnet transactions (January 2026):

| Component | Size | Cost (@1000 sat/byte) |
|-----------|------|----------------------|
| Commit TX (base) | ~300 bytes | ~0.003 RXD |
| Reveal TX (base) | ~250 bytes | ~0.0025 RXD |
| Thumbnail (150px, 65%) | ~6,000 bytes | ~0.06 RXD |
| Thumbnail (200px, 70%) | ~15,000 bytes | ~0.15 RXD |
| CBOR metadata | ~200-500 bytes | ~0.002-0.005 RXD |

**Total Cost Formula:**
```
Total = Commit Fee + Reveal Fee + (Thumbnail Size × 0.00001 RXD/byte)
```

**Example with 150px thumbnail:**
- Commit: 0.003 RXD
- Reveal base: 0.0025 RXD
- Thumbnail (6KB): 0.06 RXD
- Metadata (300 bytes): 0.003 RXD
- **Total: ~0.07 RXD**

**Example with 200px thumbnail:**
- Commit: 0.003 RXD
- Reveal base: 0.0025 RXD
- Thumbnail (15KB): 0.15 RXD
- Metadata (300 bytes): 0.003 RXD
- **Total: ~0.16 RXD**

---

## IPFS Integration

### Purpose

Use IPFS for full-resolution images while keeping on-chain thumbnails small:
- **On-chain (`main` field):** Small thumbnail for wallet display
- **IPFS (`loc` field):** Full-resolution original

### Server-Side IPFS Upload (Pinata)

```php
function uploadFileToPinata($fileData, $filename, $mimeType = 'image/jpeg') {
    $jwt = getenv('PINATA_JWT');

    if (empty($jwt)) {
        return ['success' => false, 'error' => 'Pinata JWT not configured'];
    }

    // Handle base64 data URL
    if (strpos($fileData, 'data:') === 0) {
        $parts = explode(',', $fileData, 2);
        if (count($parts) === 2) {
            $fileData = base64_decode($parts[1]);
        }
    }

    $tmpFile = tempnam(sys_get_temp_dir(), 'pinata_');
    file_put_contents($tmpFile, $fileData);

    $cfile = new CURLFile($tmpFile, $mimeType, $filename);

    $ch = curl_init('https://api.pinata.cloud/pinning/pinFileToIPFS');
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $jwt],
        CURLOPT_POSTFIELDS => ['file' => $cfile],
        CURLOPT_TIMEOUT => 60
    ]);

    $response = curl_exec($ch);
    curl_close($ch);
    unlink($tmpFile);

    $result = json_decode($response, true);
    $gateway = getenv('PINATA_GATEWAY') ?: 'gateway.pinata.cloud';

    return [
        'success' => true,
        'cid' => $result['IpfsHash'],
        'url' => 'ipfs://' . $result['IpfsHash'],
        'gatewayUrl' => "https://{$gateway}/ipfs/" . $result['IpfsHash']
    ];
}
```

### SSL Certificate Issues

**Problem:** "SSL certificate problem: unable to get local issuer certificate"

**Development Fix:** Add to `.env`:
```bash
IPFS_SKIP_SSL_VERIFY=true
```

**Production:** Configure proper CA bundle path in `curl.cainfo` php.ini setting.

---

## Common Errors & Solutions

### "NFT shows as blank card in wallet"

**Cause:** Missing `main` field with on-chain image data.

**Fix:** Add thumbnail to payload:
```javascript
payload.main = {
    t: 'image/jpeg',
    b: thumbnailUint8Array  // Must be Uint8Array, not base64
};
```

### "NFT shows as 'Unknown NFT' with no metadata"

**Cause:** NFT was encoded with JSON instead of CBOR.

**Symptoms:**
- NFT appears but shows as "Unknown NFT"
- No attributes visible
- Browser console shows: "CBOR library not loaded, using JSON fallback"

**Fix:**
1. Download CBOR library: `curl -o js/cbor.min.js "https://raw.githubusercontent.com/paroga/cbor-js/master/cbor.js"`
2. Load BEFORE blockchain scripts in HTML
3. Verify: `console.log(typeof CBOR)` should output "object"
4. Mint new NFT (old one cannot be fixed)

### "Extra items left on stack after execution"

**Cause:** Using P2PKH commit instead of nftCommitScript.

**Fix:** Pass `glyphHex` to commit transaction:
```javascript
const glyphData = encodeGlyphData(payload);
const glyphHex = Array.from(glyphData).map(b => b.toString(16).padStart(2, '0')).join('');
const commitResult = await createCommitTransaction(feeRate, glyphHex);
```

### "Unable to sign input, invalid stack size"

**Cause:** Wallet RPC cannot sign custom nftCommitScript.

**Fix:** Use external signing with Node.js and radiantjs (see Signing Challenge section).

### "min relay fee not met"

**Cause:** Fee calculation error.

**Fix:**
```php
// Radiant uses 1000 sat/byte (NOT sat/kB)
$feeSats = $txSize * 1000 * 1.5;  // With safety margin
```

### "reference-operations" error

**Cause:** Incorrect ref format or extra push opcode.

**Fix:**
- Verify ref = reversed(txid) + LE(vout), exactly 36 bytes
- Use `d8<ref>75...` NOT `d824<ref>75...`

### "Signature must be zero"

**Cause:** Signing with P2PKH subscript instead of full commit script.

**Fix:** In signing script, use `output.script` (full nftCommitScript).

### "SSL certificate problem" (IPFS upload)

**Cause:** Missing or misconfigured CA certificates.

**Development Fix:**
```bash
# In .env
IPFS_SKIP_SSL_VERIFY=true
```

**Production Fix:** Configure proper SSL certificates in PHP.

---

## Complete Implementation Example

### Full Minting Flow with Thumbnail

```javascript
class GlyphNFTMinter {
    async mintNFT(imageDataUrl, metadata, ownerAddress) {
        // Step 1: Create thumbnail for on-chain storage (225px @ 90% WebP)
        const thumbnail = await this.createThumbnail(imageDataUrl, 225, 0.90);
        console.log(`Thumbnail: ${thumbnail.length} bytes`);

        // Step 2: Upload full-res to IPFS (optional)
        const ipfsResult = await this.uploadToIPFS(imageDataUrl);
        console.log(`IPFS: ${ipfsResult.url}`);

        // Step 3: Build payload with main field
        const payload = {
            p: [2],
            name: metadata.name,
            type: metadata.type || 'photo',
            main: {
                t: 'image/webp',  // WebP for best quality/size ratio
                b: thumbnail
            },
            loc: ipfsResult.url,
            attrs: metadata.attrs || {}
        };

        // Step 4: Encode to CBOR
        const glyphData = this.encodeGlyphData(payload);
        const glyphHex = Array.from(glyphData)
            .map(b => b.toString(16).padStart(2, '0'))
            .join('');

        // Step 5: Create commit transaction
        const commitResult = await this.createCommitTransaction(glyphHex);
        console.log('Commit TX:', commitResult.txid);

        // Step 6: Wait for confirmation
        await this.waitForConfirmation(commitResult.txid);

        // Step 7: Create and sign reveal transaction
        const revealResult = await this.createRevealTransaction(
            commitResult, glyphHex, ownerAddress
        );
        console.log('Reveal TX:', revealResult.txid);

        return {
            commitTxid: commitResult.txid,
            revealTxid: revealResult.txid,
            glyphId: `${revealResult.txid}:0`,
            ref: revealResult.ref,
            thumbnailSize: thumbnail.length,
            ipfsUrl: ipfsResult.url
        };
    }

    async createThumbnail(dataUrl, maxSize = 225, quality = 0.90) {
        return new Promise((resolve, reject) => {
            const img = new Image();
            img.onload = () => {
                let width = img.width;
                let height = img.height;

                if (width > height) {
                    if (width > maxSize) {
                        height = Math.round((height * maxSize) / width);
                        width = maxSize;
                    }
                } else {
                    if (height > maxSize) {
                        width = Math.round((width * maxSize) / height);
                        height = maxSize;
                    }
                }

                const canvas = document.createElement('canvas');
                canvas.width = width;
                canvas.height = height;
                const ctx = canvas.getContext('2d');
                ctx.imageSmoothingEnabled = true;
                ctx.imageSmoothingQuality = 'high';
                ctx.drawImage(img, 0, 0, width, height);

                // Use WebP for best quality/size ratio
                canvas.toBlob((blob) => {
                    const reader = new FileReader();
                    reader.onload = () => resolve(new Uint8Array(reader.result));
                    reader.onerror = reject;
                    reader.readAsArrayBuffer(blob);
                }, 'image/webp', quality);
            };
            img.onerror = reject;
            img.src = dataUrl;
        });
    }

    encodeGlyphData(data) {
        if (typeof CBOR === 'undefined') {
            throw new Error('CBOR library not loaded!');
        }
        if (!data.p) data.p = [2];

        const marker = new TextEncoder().encode('gly');
        const cborData = CBOR.encode(data);
        const payload = new Uint8Array(cborData);

        const result = new Uint8Array(marker.length + payload.length);
        result.set(marker, 0);
        result.set(payload, marker.length);
        return result;
    }
}
```

---

## Testing & Verification

### Verify CBOR Encoding

```javascript
// Before minting, verify CBOR is working
const testPayload = { p: [2], name: "Test" };
const encoded = CBOR.encode(testPayload);
const decoded = CBOR.decode(encoded);
console.log('CBOR test:', decoded.name === "Test" ? 'PASS' : 'FAIL');
```

### Verify Thumbnail

```javascript
// Check thumbnail size before minting
const thumbnail = await createThumbnail(imageDataUrl, 150, 0.65);
console.log(`Thumbnail size: ${thumbnail.length} bytes`);
if (thumbnail.length > 10000) {
    console.warn('Thumbnail large - consider reducing quality');
}
```

### Verify on Glyph Explorer

Visit: `https://glyph.photonic.online/tx/<reveal_txid>`

You should see:
- NFT image displayed (from `main` field)
- NFT metadata
- Container/author refs (if used)
- Attributes

### Verify in Glyphium Wallet

Import your wallet and check:
- NFT appears in collection
- Thumbnail is visible and clear
- Attributes are readable

### Verified Working Transactions (January 2026)

**With on-chain thumbnail:**
- Reveal: `27390efab1e3168c05301b18f6cdfd553a6d122a41496d0f5e104e79a918be7e`
- Thumbnail: 150x200, ~14KB, displays correctly in Glyphium

---

## Appendix: Quick Reference

### Opcodes

| Hex | Name | Notes |
|-----|------|-------|
| `aa` | OP_HASH256 | Double SHA256 |
| `88` | OP_EQUALVERIFY | Check equal and remove |
| `c0` | OP_INPUTINDEX | BCH introspection |
| `c8` | OP_OUTPOINTTXHASH | BCH introspection - input txid |
| `c9` | OP_OUTPOINTINDEX | BCH introspection - input vout |
| `da` | OP_REFTYPE_OUTPUT | Check ref type in outputs |
| `d8` | OP_PUSHINPUTREFSINGLETON | Create singleton ref |
| `75` | OP_DROP | Remove top stack item |
| `76` | OP_DUP | Duplicate top stack item |
| `a9` | OP_HASH160 | RIPEMD160(SHA256(x)) |
| `ac` | OP_CHECKSIG | Verify signature |

### Key Hex Values

| Purpose | Hex |
|---------|-----|
| "gly" marker | `676c79` |
| Push 3 bytes | `03` |
| Push 20 bytes | `14` |
| Push 32 bytes | `20` |
| OP_PUSHDATA1 | `4c` |

### Checklist Before Minting

- [ ] CBOR library loaded (`typeof CBOR === 'object'`)
- [ ] Thumbnail created (Uint8Array, < 15KB recommended)
- [ ] `main` field added to payload with `t` and `b` properties
- [ ] Protocol set (`p: [2]`)
- [ ] Name set
- [ ] Sufficient RXD balance for fees

### Cost Quick Reference

| Thumbnail | Format | Approx. Size | Approx. Total Cost |
|-----------|--------|--------------|-------------------|
| 100px, 50% | JPEG | 2-4 KB | ~0.04 RXD |
| 150px, 65% | JPEG | 5-8 KB | ~0.07 RXD |
| 200px, 85% | WebP | 15-17 KB | ~0.17 RXD |
| **225px, 90%** | **WebP** | **20-25 KB** | **~0.22 RXD** |
| 300px, 80% | JPEG | 20-30 KB | ~0.30 RXD |

---

**Last Updated:** 2026-01-11
**Based on Verified Mainnet Transactions:**
- With thumbnail: `27390efab1e3168c05301b18f6cdfd553a6d122a41496d0f5e104e79a918be7e`

**Critical Discoveries Documented:**
1. On-chain images (`main` field) required for wallet display
2. CBOR encoding mandatory (JSON = "Unknown NFT")
3. Optimal thumbnail: 225px WebP @ 90% quality (~22KB, ~0.22 RXD) for best quality/cost balance

**License:** MIT
**Radiant Blockchain:** https://radiantblockchain.org
