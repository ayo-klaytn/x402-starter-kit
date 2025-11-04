# Codebase Breakdown: x402 Payment System for Kaia Kairos

This document explains how the x402 payment system works and how it's configured for Kaia Kairos testnet.

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Key Components](#key-components)
3. [Kaia Kairos Configuration](#kaia-kairos-configuration)
4. [Payment Flow](#payment-flow)
5. [Environment Variables](#environment-variables)
6. [What We Changed](#what-we-changed)

---

## 🏗️ Architecture Overview

The x402 payment system implements a **pay-per-request** model where clients must sign EIP-3009 payment authorizations before accessing AI services.

```
┌─────────────┐
│   Client    │ (Signs payment with wallet)
└──────┬──────┘
       │ HTTP POST /process
       ▼
┌─────────────────┐
│  Express Server │ (server.ts)
│  - Validates    │
│  - Verifies     │
│  - Processes    │
└──────┬──────────┘
       │
       ├──► MerchantExecutor (verifies & settles payments)
       │
       └──► ExampleService (processes AI requests)
```

### Two Settlement Modes

1. **Facilitator Mode** (default): Uses x402.org facilitator service
   - Facilitator handles RPC calls and blockchain transactions
   - No local RPC or private key needed
   - Works out of the box

2. **Direct Mode** (`SETTLEMENT_MODE=local`): Direct on-chain settlement
   - Requires `PRIVATE_KEY` and `RPC_URL`
   - Server directly calls `transferWithAuthorization()` on-chain
   - More control, but requires gas management

---

## 🔧 Key Components

### 1. **server.ts** - Main Express Server

**Purpose**: HTTP API server that handles payment-required requests

**Key Responsibilities**:
- Loads environment configuration
- Initializes `MerchantExecutor` and `ExampleService`
- Exposes `/health` and `/process` endpoints
- Handles payment verification flow

**Key Code Sections**:

```typescript
// Network configuration (lines 54-70)
const SUPPORTED_NETWORKS: Network[] = [
  // ... built-in networks
  'kaia' as Network,           // Kaia mainnet
  'kaia-kairos' as Network,    // Kaia testnet ← ADDED
];

// Merchant executor setup (lines 148-163)
const merchantOptions: MerchantExecutorOptions = {
  payToAddress: PAY_TO_ADDRESS,
  network: resolvedNetwork,      // 'kaia-kairos'
  price: 0.1,
  assetAddress: ASSET_ADDRESS,   // Your token address
  assetName: ASSET_NAME,         // Token name (must match ERC20 name())
  assetVersion: ASSET_VERSION,  // EIP-712 version ("2")
  chainId: CHAIN_ID,            // 1001 for Kairos
  // ... settlement mode config
};
```

**Payment Flow in `/process` endpoint** (lines 205-363):

1. **Check for payment** (line 246-249): Looks for `x402.payment.payload` in message metadata
2. **If no payment** (line 251): Returns 402 Payment Required response
3. **If payment exists** (line 288): Verifies signature using `MerchantExecutor`
4. **If valid** (line 330): Processes request via `ExampleService`
5. **Settles payment** (line 332): Calls `transferWithAuthorization()` on-chain

---

### 2. **MerchantExecutor.ts** - Payment Handler

**Purpose**: Handles EIP-3009 payment verification and on-chain settlement

**Key Responsibilities**:
- Creates payment requirements (amount, token, network)
- Verifies EIP-712 signatures locally
- Settles payments on-chain via `transferWithAuthorization()`
- Supports both facilitator and direct modes

**EIP-3009 Structure** (lines 10-19):

```typescript
const TRANSFER_AUTH_TYPES = {
  TransferWithAuthorization: [
    { name: 'from', type: 'address' },
    { name: 'to', type: 'address' },
    { name: 'value', type: 'uint256' },
    { name: 'validAfter', type: 'uint256' },
    { name: 'validBefore', type: 'uint256' },
    { name: 'nonce', type: 'bytes32' },
  ],
};
```

**EIP-712 Domain Building** (lines 579-586):

```typescript
private buildEip712Domain(requirements: PaymentRequirements) {
  return {
    name: requirements.extra?.name || this.assetName,      // Token name
    version: requirements.extra?.version || this.assetVersion || '2',  // "2"
    chainId: this.chainId,                                 // 1001 for Kairos
    verifyingContract: requirements.asset,                  // Token address
  };
}
```

**Payment Verification** (lines 352-457):
- Validates network match
- Validates recipient address
- Validates amount
- Validates timestamps (validAfter/validBefore)
- Verifies EIP-712 signature using `ethers.verifyTypedData()`

**On-Chain Settlement** (lines 459-534):
- Splits signature into `v`, `r`, `s` components
- Calls `transferWithAuthorization()` on token contract
- Waits for transaction receipt

---

### 3. **testClient.ts** - Test Client

**Purpose**: Client-side code that signs payments and submits requests

**Key Responsibilities**:
- Signs EIP-3009 authorizations using wallet
- Submits payment payloads to server
- Demonstrates complete payment flow

**Chain ID Mapping** (lines 32-41):

```typescript
const CHAIN_IDS: Record<string, number> = {
  // ... other networks
  kaia: 8217,           // Kaia mainnet
  'kaia-kairos': 1001,  // Kaia testnet ← ADDED
};
```

**Payment Signing** (lines 63-98):

1. Creates authorization object (from, to, value, timestamps, nonce)
2. Builds EIP-712 domain (name, version "2", chainId 1001, token address)
3. Signs using `wallet.signTypedData()`
4. Returns payment payload

---

### 4. **ExampleService.ts** - AI Service

**Purpose**: Processes actual service requests (AI in this case)

**Note**: This is just an example - replace with your own service logic.

---

## 🌐 Kaia Kairos Configuration

### Environment Variables Required

```env
# Network Configuration
NETWORK=kaia-kairos              # Network identifier
CHAIN_ID=1001                     # Kaia Kairos chain ID

# Token Configuration
ASSET_ADDRESS=0x38B730Cda4a7da25110e8bFaF56C2F804fB69B9a  # Your ERC-3009 token
ASSET_NAME="Your Token Name"      # Must match ERC20 name() exactly
ASSET_VERSION=2                   # EIP-712 domain version

# Payment Configuration
PAY_TO_ADDRESS=0x...              # Address that receives payments

# Settlement Configuration (for direct mode)
SETTLEMENT_MODE=local             # Use direct on-chain settlement
PRIVATE_KEY=0x...                 # Wallet with gas for settlement
RPC_URL=https://...               # Kaia Kairos RPC endpoint

# Optional
EXPLORER_URL=https://baobab.klaytnscope.com  # Block explorer
```

### Why These Settings Matter

1. **ASSET_NAME**: Must match the exact string returned by your token's `name()` function
   - EIP-712 domain includes the token name
   - Mismatch = "Invalid signature" error

2. **ASSET_VERSION**: Must match your token's EIP-712 domain version
   - Your contract uses `"2"` (matches Circle USDC standard)
   - Set via `ASSET_VERSION=2` or defaults to `"2"`

3. **CHAIN_ID**: Must be `1001` for Kaia Kairos
   - Used in EIP-712 domain separator
   - Prevents signature replay across networks

4. **ASSET_ADDRESS**: Your deployed ERC-3009 token contract
   - Must implement `transferWithAuthorization()` function
   - Must use EIP-712 with version "2"

---

## 🔄 Payment Flow

### Complete Flow Diagram

```
┌──────────────┐
│   Client     │
└──────┬───────┘
       │ 1. POST /process (no payment)
       ▼
┌─────────────────┐
│   Server        │
│  server.ts      │
└──────┬──────────┘
       │ 2. Returns 402 Payment Required
       │    { accepts: [{ network: "kaia-kairos", asset: "0x...", ... }] }
       ▼
┌──────────────┐
│   Client     │
│ testClient.ts│
└──────┬───────┘
       │ 3. Signs EIP-3009 authorization
       │    - Domain: { name, version: "2", chainId: 1001, verifyingContract }
       │    - Data: { from, to, value, validAfter, validBefore, nonce }
       │    - Signature: wallet.signTypedData()
       ▼
┌──────────────┐
│   Client     │
└──────┬───────┘
       │ 4. POST /process (with payment payload)
       ▼
┌─────────────────┐
│   Server        │
│ MerchantExecutor│
└──────┬──────────┘
       │ 5. verifyPayment()
       │    - Verifies EIP-712 signature locally
       │    - Validates amounts, timestamps, network
       ▼
┌─────────────────┐
│   Server        │
│ ExampleService  │
└──────┬──────────┘
       │ 6. execute() - Processes request
       ▼
┌─────────────────┐
│   Server        │
│ MerchantExecutor│
└──────┬──────────┘
       │ 7. settlePayment()
       │    - Calls transferWithAuthorization() on-chain
       │    - Splits signature: v, r, s
       │    - Waits for transaction receipt
       ▼
┌──────────────┐
│   Client     │
└──────────────┘
       │ 8. Receives response + settlement tx hash
```

### Step-by-Step Breakdown

#### Step 1: Client Sends Request (No Payment)
```typescript
// testClient.ts - sendRequest()
POST /process
{
  "message": {
    "parts": [{"kind": "text", "text": "Hello!"}]
  }
}
```

#### Step 2: Server Returns Payment Requirements
```typescript
// server.ts - createPaymentRequiredResponse()
Response: {
  "error": "Payment Required",
  "task": {
    "metadata": {
      "x402.payment.required": {
        "x402Version": 1,
        "accepts": [{
          "scheme": "eip3009",
          "network": "kaia-kairos",
          "asset": "0x38B730Cda4a7da25110e8bFaF56C2F804fB69B9a",
          "payTo": "0x...",
          "maxAmountRequired": "100000",  // $0.10 in micro units
          "extra": {
            "name": "Your Token Name",
            "version": "2"
          }
        }]
      }
    }
  }
}
```

#### Step 3: Client Signs Payment
```typescript
// testClient.ts - createPaymentPayload()
const domain = {
  name: "Your Token Name",      // From ASSET_NAME
  version: "2",                  // From ASSET_VERSION
  chainId: 1001,                 // Kaia Kairos
  verifyingContract: "0x38B730..." // From ASSET_ADDRESS
};

const authorization = {
  from: wallet.address,
  to: "0x...",                   // PAY_TO_ADDRESS
  value: "100000",
  validAfter: "0",
  validBefore: "1735689600",
  nonce: "0x..."
};

const signature = await wallet.signTypedData(domain, TRANSFER_AUTH_TYPES, authorization);
```

#### Step 4: Client Submits Payment
```typescript
// testClient.ts - sendPaidRequest()
POST /process
{
  "message": {
    "parts": [{"kind": "text", "text": "Hello!"}],
    "metadata": {
      "x402.payment.payload": {
        "x402Version": 1,
        "scheme": "eip3009",
        "network": "kaia-kairos",
        "payload": {
          "authorization": { ... },
          "signature": "0x..."
        }
      },
      "x402.payment.status": "payment-submitted"
    }
  }
}
```

#### Step 5: Server Verifies Payment
```typescript
// MerchantExecutor.ts - verifyPaymentLocally()
1. Checks network match
2. Checks recipient match
3. Checks amount sufficient
4. Checks timestamps valid
5. Verifies signature using ethers.verifyTypedData()
   - Rebuilds same domain
   - Recovers signer address
   - Compares to authorization.from
```

#### Step 6: Server Processes Request
```typescript
// ExampleService.ts - execute()
// Processes AI request, generates response
```

#### Step 7: Server Settles Payment On-Chain
```typescript
// MerchantExecutor.ts - settleOnChain()
const parsedSignature = ethers.Signature.from(signature);

await usdcContract.transferWithAuthorization(
  authorization.from,
  authorization.to,
  authorization.value,
  authorization.validAfter,
  authorization.validBefore,
  authorization.nonce,
  parsedSignature.v,  // Split signature
  parsedSignature.r,
  parsedSignature.s
);
```

#### Step 8: Server Returns Response
```typescript
Response: {
  "success": true,
  "task": { ... },  // AI response
  "settlement": {
    "success": true,
    "transaction": "0x...",  // On-chain tx hash
    "network": "kaia-kairos"
  }
}
```

---

## 🔧 What We Changed

### 1. Added Kaia Networks to Server (server.ts)

**File**: `src/server.ts` (lines 67-69)

```typescript
const SUPPORTED_NETWORKS: Network[] = [
  // ... existing networks
  'kaia' as Network,           // ← ADDED
  'kaia-kairos' as Network,     // ← ADDED
];
```

**Why**: Allows the server to accept `NETWORK=kaia-kairos` as a valid network identifier.

---

### 2. Added Kaia Chain IDs to Test Client (testClient.ts)

**File**: `src/testClient.ts` (lines 38-40)

```typescript
const CHAIN_IDS: Record<string, number> = {
  // ... existing networks
  kaia: 8217,           // ← ADDED
  'kaia-kairos': 1001,  // ← ADDED
};
```

**Why**: Enables the test client to sign EIP-712 messages with the correct chain ID (1001 for Kairos).

---

### 3. Added ASSET_VERSION Support (MerchantExecutor.ts & server.ts)

**Files**: 
- `src/MerchantExecutor.ts` (lines 130, 159, 185, 202, 586)
- `src/server.ts` (lines 34, 160)

**Changes**:
- Added `assetVersion` to `MerchantExecutorOptions`
- Added `ASSET_VERSION` environment variable support
- Defaults to `"2"` if not provided (matches your token)

**Why**: Your token uses EIP-712 version `"2"`, so we need to explicitly set this in the domain.

---

### 4. Added Debug Logging (MerchantExecutor.ts)

**File**: `src/MerchantExecutor.ts` (lines 424-428, 490-494, 515-519)

**Changes**: Added console logs to show:
- EIP-712 domain parameters (name, version, chainId, verifyingContract)
- Signature components (v, r, s)

**Why**: Helps debug signature mismatches by showing exactly what domain is being used.

---

## 📝 Key Takeaways

1. **EIP-712 Domain Must Match Exactly**
   - Token name from `ASSET_NAME` must match `ERC20.name()`
   - Version must match token's EIP-712 version (usually "2")
   - Chain ID must match network (1001 for Kairos)
   - Contract address must be correct

2. **Signature Format**
   - Client signs with `wallet.signTypedData()`
   - Server splits signature into `v`, `r`, `s` for on-chain call
   - Uses `ethers.Signature.from()` to parse

3. **Payment Flow is Two-Step**
   - First request: Returns payment requirements
   - Second request: Includes signed payment payload

4. **Settlement Modes**
   - **Facilitator**: Easiest, no RPC needed
   - **Direct**: More control, requires RPC + private key

5. **For Custom Networks Like Kaia**
   - Add network name to `SUPPORTED_NETWORKS`
   - Add chain ID to `CHAIN_IDS` in test client
   - Provide `ASSET_ADDRESS`, `ASSET_NAME`, `ASSET_VERSION`, `CHAIN_ID` in env
   - Ensure token implements EIP-3009 with EIP-712 version matching

---

## 🚀 Next Steps

1. **Replace ExampleService**: Replace `src/ExampleService.ts` with your actual service logic
2. **Customize Payment Requirements**: Adjust price, timeout, etc. in `MerchantExecutor` initialization
3. **Add More Networks**: Follow the same pattern to add other EVM networks
4. **Production Deployment**: Use facilitator mode for production, or set up your own facilitator service

---

## 📚 References

- [EIP-3009: Transfer With Authorization](https://eips.ethereum.org/EIPS/eip-3009)
- [EIP-712: Typed Structured Data Hashing and Signing](https://eips.ethereum.org/EIPS/eip-712)
- [x402 Protocol](https://x402.org/)
- [Kaia Network Documentation](https://docs.kaia.io/)

