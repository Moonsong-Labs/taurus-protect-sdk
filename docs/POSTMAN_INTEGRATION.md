# Postman Integration

This document describes how to use the Taurus-PROTECT Postman collections to explore and test the API. These collections provide a ready-to-use environment for both HMAC and Bearer authentication workflows.

> **Note:** These collections are intended to supplement our API documentation and training materials. This collection is examples of how to achieve a variety of workflows however are not intended to be used in production as Postman is primarily a development tool.

---

## Overview

Two Postman collections are included in the `postman/` directory:

| Collection | File | Authentication |
|------------|------|----------------|
| Bearer Authentication | `Bearer Authentication.postman_collection.json` | OAuth-style Bearer token |
| HMAC Authentication | `Hmac Based Authentication.postman_collection.json` | TPV1-HMAC-SHA256 (same scheme as the SDKs) |

Both collections cover the same set of API endpoints and each collection provides complete examples with the respective authentication type.

- **Bearer Authentication** requires obtaining a short-lived token via a login endpoint before making requests.
- **HMAC Authentication** uses the same TPV1-HMAC-SHA256 signing scheme as the SDKs — a pre-request script computes the `Authorization` header automatically for every request.

These collections are intentionally minimal: they cover core endpoints without exhaustive query parameters, providing a solid foundation of examples to accompany our APIs.

---

## Collection Structure

Both collections are organized into the same set of folders:

| Folder | Endpoints |
|--------|-----------|
| **Wallet & Addresses** | List Addresses, List Currencies, List Wallets, Create a Wallet, Create an Address |
| **Whitelist Address** | List Whitelisted Addresses, Create a Whitelisted Address, List Whitelisted Addresses for Approval, Approve a Whitelisted Address, Delete a Whitelisted Address |
| **Transaction** | Create an Outgoing Request, List Requests for Approval, Approve Request |
| **Users** | Get User, Update User |
| **Changes** | List Changes for Approval, List Changes, Approve Changes, Create a Change |
| **Prices** | List Prices, Update a Price |
| **Staking (Solana)** | List Whitelisted Validators, List SOL Wallets, List Wallet Addresses, Create a SOL Stake Request, Get Staking Request Details, Approve Staking Request |

> **Signing required:** `Approve Request` and `Update a Price` require an ECDSA signature in addition to standard authentication. See [Request Signing](#request-signing-ecdsa) below.

---

## Prerequisites

### Postman Version

Any recent version of Postman (desktop or web) that supports:
- Environment variables
- Postman Vault
- Pre-request scripts (JavaScript)

### Postman Vault Setup

Postman Vault stores sensitive values that are never exposed in collection exports or version control. Configure it before using either collection.

1. Open Postman → click the **Settings** icon (gear) → select the **Vault** tab
2. Add the following secrets:

| Secret | Required For | Description |
|--------|-------------|-------------|
| `userPrivateKey` | Both collections (signing) | Your EC private key in PEM format — required for Approve Request and Update a Price |
| `apiKey` | HMAC collection | Your HMAC API key (UUID format) |
| `apiSecret` | HMAC collection | Your HMAC API secret (hex string) |

#### `userPrivateKey` Format

The private key must be in PEM format with both the EC parameters block and the private key block:

```
-----BEGIN EC PARAMETERS-----
BggqhkjOPQMBBw==
-----END EC PARAMETERS-----
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIBWyFiX9dTdYIrBCWJ...
-----END EC PRIVATE KEY-----
```

#### Obtaining HMAC Credentials

HMAC API keys are generated through the Admin UI:

1. Log in as an Admin user → navigate to **Users**
2. Select the target user → locate the **API Keys** section → **HMAC**
3. Click **Generate new API key**
4. A second Admin must approve the request before the key becomes active

---

## Bearer Authentication Setup

### Step 1: Create Environments

Create two Postman environments — one for a regular user and one for an Admin user.

**Environment: User**

| Variable | Example Value | Description |
|----------|---------------|-------------|
| `baseUrl` | `https://YOUR-INSTANCE.t-dx.com` | Your Taurus-PROTECT instance URL |
| `username` | `user01@example.com` | User account email |
| `password` | `your-password` | User account password |
| `authToken` | *(auto-populated)* | Populated automatically after Get Token |

**Environment: Admin**

| Variable | Example Value | Description |
|----------|---------------|-------------|
| `baseUrl` | `https://YOUR-INSTANCE.t-dx.com` | Your Taurus-PROTECT instance URL |
| `adminUsername` | `admin01@example.com` | Admin account email |
| `adminPassword` | `admin-password` | Admin account password |
| `authToken` | *(auto-populated)* | Populated automatically after Get Token |

### Step 2: Select an Environment

Use the environment dropdown in Postman to select either the User or Admin environment before sending requests.

### Step 3: Authenticate

The collection contains two authentication requests at the top level:

- `[Click this first] (User) Bearer Authentication - Get Token`
- `[Click this first] (Admin) Bearer Authentication - Get Token`

Both call `POST /api/rest/v1/authentication/token`. The difference is which credentials they read:
- The User request reads `username` and `password`
- The Admin request reads `adminUsername` and `adminPassword`

Each request includes a post-response script that extracts the token from the response and saves it to the `authToken` environment variable. All other requests in the collection reference `{{authToken}}` automatically.

> **Token expiry:** Bearer tokens are valid for 30 minutes. Re-run the Get Token request when your token expires.

---

## HMAC Authentication Setup

### How It Works

The HMAC collection includes a pre-request script at the **collection level**. This script runs automatically before every request and computes the TPV1-HMAC-SHA256 `Authorization` header — the same scheme used by the SDKs. See [Authentication & Security](AUTHENTICATION.md) for a full description of the TPV1 signing algorithm.

### Setup

1. Add `apiKey` and `apiSecret` to **Postman Vault** (see [Prerequisites](#prerequisites) above)
2. Set the `baseUrl` environment variable to your instance URL
3. Send any request — the pre-request script handles authentication automatically

No separate "Get Token" step is required. Every request is independently signed.

### Skipped Endpoints

The collection-level pre-request script intentionally skips HMAC signing for two endpoints:

| Endpoint | Reason |
|----------|--------|
| `POST /api/rest/v1/requests/approve` | Requires an ECDSA signature to be computed *before* the HMAC signature |
| `PUT /api/rest/v1/prices` | Requires an ECDSA signature to be computed *before* the HMAC signature |
| `PUT /api/rest/v1/whitelists/addresses/approve` | Requires an ECDSA signature to be computed *before* the HMAC signature |

These endpoints have their own per-request pre-request scripts that handle ECDSA signing and then compute the HMAC header in the correct order.

---

## Request Signing (ECDSA)

### Affected Endpoints

Two endpoints require an ECDSA signature over the request payload:

- **Approve Request** (`POST /api/rest/v1/requests/{id}/approve`)
- **Update a Price** (`PUT /api/rest/v1/prices`)

### What Gets Signed

For request approvals, the user signs the `metadata.hash` field — a SHA-256 hash of the transaction details (source address, destination address, amount, currency, and other parameters). This ensures the approving user cryptographically attests to the exact transaction details. See [Authentication & Security](AUTHENTICATION.md#request-approval-signing) for more detail.

### How the Pre-request Script Handles It

When you send one of these requests, the per-request pre-request script:

1. Retrieves `userPrivateKey` from Postman Vault
2. Computes the ECDSA P-256 signature over the relevant payload
3. Constructs the request body including the signature
4. In the HMAC collection: then computes the HMAC `Authorization` header over the complete body

The script handles all cryptographic operations automatically once `userPrivateKey` is set in Vault.

### Required Environment Variables

Each signing endpoint documents its required environment variables in its own pre-request script. Review the script comments for the specific variables needed before sending.

---

## Troubleshooting

### Body Not Set / Authorization Header Missing

Postman can occasionally fail to apply the body or `Authorization` header computed by a pre-request script. If a request fails unexpectedly:

1. Send the request a second time
2. If it still fails, open the Postman console (`View → Show Postman Console`), add a `console.log` to inspect the computed body or header, then manually paste the value into the **Body** or **Headers** tab before sending

---

## Related Documentation

- [Authentication & Security](AUTHENTICATION.md) — TPV1-HMAC-SHA256 protocol, ECDSA signing, credential management
- [Key Concepts](CONCEPTS.md) — Domain model: wallets, addresses, requests, transactions
- [Integrity Verification](INTEGRITY_VERIFICATION.md) — Cryptographic verification flows for governance rules
