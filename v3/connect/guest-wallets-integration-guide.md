# HandCash Connect — Guest Wallets Integration Guide

This guide is for **Connect app developers** who want to create **guest wallets** for users inside their application, operate those wallets with `@handcash/sdk`, and later send users to HandCash to **claim a permanent handle**.

Guest wallets are created via **Connect REST endpoints** (server-side). Wallet operations use **`@handcash/sdk`**. Handle claim happens on **HandCash** (consumer web flow).

---

## Table of contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Integration flow](#integration-flow)
5. [Step 1 — Request email verification code](#step-1--request-email-verification-code)
6. [Step 2 — Generate access keys](#step-2--generate-access-keys)
7. [Step 3 — Submit email OTP](#step-3--submit-email-otp)
8. [Step 4 — Create guest account](#step-4--create-guest-account)
9. [Step 5 — Operate the wallet with the SDK](#step-5--operate-the-wallet-with-the-sdk)
10. [Detecting guest vs full accounts](#detecting-guest-vs-full-accounts)
11. [Upgrade — claim a permanent handle](#upgrade--claim-a-permanent-handle)
12. [Existing HandCash users (Connect auth)](#existing-handcash-users-connect-auth)
13. [Error handling](#error-handling)
14. [Security requirements](#security-requirements)
15. [Reference implementation](#reference-implementation)
16. [Testing checklist](#testing-checklist)
17. [FAQ](#faq)
18. [Support](#support)

---

## Overview

### What is a guest wallet?

A guest wallet is a **real HandCash wallet** created programmatically for your app user:

- Auto-assigned handle: `guest000001`, `guest000002`, …
- Verified email
- Your app is **authorized immediately** after creation
- Same wallet keys, balance, and items persist if the user later claims a permanent handle

### What this guide covers

| Topic | Method |
|-------|--------|
| Create guest wallet | Connect REST (`/v3/connect/account/*`) from your **backend** |
| Pay, balance, items | `@handcash/sdk` with stored `authToken` |
| Claim permanent handle | Redirect user to **HandCash web** (same email) |
| Link existing HC user | Connect authorization redirect (`getRedirectionUrl`) |

### What is not included

- **`Connect.createUserAccount()`** — this method does not exist in `@handcash/sdk`. Ignore any docs that reference it.
- **Client-side account creation with `app-secret`** — secrets must stay on your server.

---

## Prerequisites

Before integrating, ensure you have:

| Requirement | Details |
|-------------|---------|
| **App credentials** | `app-id` and `app-secret` from [Developer Dashboard](https://dashboard.handcash.io) |
| **Feature flag** | HandCash must enable `createAccounts` on your App ID. Contact [Support@handcash.io](mailto:Support@handcash.io) or your HandCash contact before production. |
| **Backend server** | Node.js (or any server) to call Connect REST with `app-secret` |
| **HTTPS** | Required in production for redirects and API calls |
| **Callback URLs** | Authorization Success / Failed URLs configured in dashboard (needed for existing-user Connect flow) |

Install the SDK on your server:

```bash
npm install @handcash/sdk
```

Optional key generation dependency (if not using another secp256k1 library):

```bash
npm install @noble/secp256k1
```

---

## Architecture

```
┌──────────────┐   app-id + app-secret   ┌─────────────────────┐
│ Your backend │ ───────────────────────►│ cloud.handcash.io   │
└──────────────┘                         │ /v3/connect/account │
       ▲                                 └─────────────────────┘
       │ authToken (private key hex)              ▲
       │                                          │ verifyEmail (server-side)
       │   requestId + OTP + accessPublicKey     │
       │ ────────────────────────────────────────┤
       │         POST /auth/verifyCode           │
┌──────────────┐ ───────────────────────────────►│ Trustholder API
│ Your client  │   (client or your backend)      │ (email OTP + key bind)
│ (browser/app)│                                 └─────────────────────┘
└──────────────┘
       │
       └── After creation: your backend uses @handcash/sdk
           (Connect.pay, Connect.getSpendableBalances, Items.*, …)
```

**Roles:**

- **Your client** — collects email and OTP, generates keypair, submits OTP verification
- **Your backend** — calls Connect REST with `app-secret`, may proxy OTP verification, stores `authToken`, runs SDK operations
- **HandCash cloud** — sends verification email, creates wallet, assigns guest handle, authorizes your app
- **Trustholder** — verifies the OTP and binds the email to `accessPublicKey` (required before account creation)

**Base URLs (production):**

| Service | URL |
|---------|-----|
| Connect (account creation, SDK) | `https://cloud.handcash.io` |
| Email OTP verification | `https://trust.hastearcade.com` |

> **Important:** `POST /v3/connect/account/` does **not** accept the OTP. You must submit the code separately in [Step 3](#step-3--submit-email-otp) before calling create account in Step 4.

---

## Integration flow

```
1. User enters email in your app
2. Your backend → POST /v3/connect/account/requestEmailCode  →  returns requestId
3. User receives OTP email and enters the code in your app
4. Client generates secp256k1 keypair (accessPublicKey + authToken)
5. Submit OTP → POST trust.hastearcade.com/auth/verifyCode
               { requestId, verificationCode, publicKey: accessPublicKey }
6. Your backend → POST /v3/connect/account/  { email, accessPublicKey }  (omit alias for guest)
7. Store authToken securely; user has guest wallet
8. Your backend uses @handcash/sdk for pay / balance / items
9. When ready → redirect user to HandCash to claim permanent handle
```

Step 5 can be called from your **client** or **backend** (see [Step 3](#step-3--submit-email-otp)). Step 6 will fail if Step 5 was skipped or the OTP was wrong.

---

## Step 1 — Request email verification code

**Your backend** calls HandCash cloud. Never expose `app-secret` to the client.

```http
POST https://cloud.handcash.io/v3/connect/account/requestEmailCode
Content-Type: application/json
app-id: YOUR_APP_ID
app-secret: YOUR_APP_SECRET

{
  "email": "user@example.com"
}
```

**Success response (200):**

```json
{
  "requestId": "abc123-request-id"
}
```

Return `requestId` to your client — it is required for [Step 3](#step-3--submit-email-otp).

**Optional:** pass `customEmailParameters` in the body to customize verification email content.

### Errors at this step

| Status | Message | Action |
|--------|---------|--------|
| 403 | `Feature not enabled - createAccounts` | Contact HandCash to enable the feature |
| 400 | `Full account exist - redirect user to login instead` | User already has a full HandCash account — use [Connect auth](#existing-handcash-users-connect-auth) instead of guest creation |
| 401 | Invalid app-id / app-secret | Check dashboard credentials |

---

## Step 2 — Generate access keys

Generate a **secp256k1 keypair** after the user enters the OTP. The same public key must be used in Steps 3 and 4.

**Node.js example (`@noble/secp256k1`):**

```typescript
import * as secp256k1 from '@noble/secp256k1';

function generateAccessKeys() {
  const privateKey = secp256k1.utils.randomPrivateKey();
  const publicKey = secp256k1.getPublicKey(privateKey);

  return {
    authToken: Buffer.from(privateKey).toString('hex'),       // store securely — never log
    accessPublicKey: Buffer.from(publicKey).toString('hex'),  // used in Steps 3 and 4
  };
}
```

| Value | Purpose |
|-------|---------|
| `accessPublicKey` | Public key hex — submitted in Step 3 (OTP verify) and Step 4 (create account) |
| `authToken` | Private key hex — sent to your backend for storage; used as `getAccountClient(authToken)` for all SDK calls |

---

## Step 3 — Submit email OTP

The user enters the OTP from the verification email. Submit it to **Trustholder** to bind the email to `accessPublicKey`. This step is **required** — account creation in Step 4 will fail without it.

```http
POST https://trust.hastearcade.com/auth/verifyCode
Content-Type: application/json

{
  "requestId": "<requestId from Step 1>",
  "verificationCode": "12345678",
  "publicKey": "<accessPublicKey hex from Step 2>"
}
```

No `app-id` or `app-secret` headers are needed for this call.

### Who calls this?

| Approach | How |
|----------|-----|
| **Client-direct** | Browser/app calls Trustholder after the user enters the OTP |
| **Backend proxy (recommended)** | Client POSTs `{ requestId, verificationCode, accessPublicKey }` to your backend; your backend calls Trustholder, then Step 4 |

A backend proxy keeps OTP submission in one place and lets you call verify + create in a single `/api/handcash/guest/create` route (see [Reference implementation](#reference-implementation)).

### Errors at this step

| Status | Action |
|--------|--------|
| 4xx | Invalid or expired OTP — let the user retry or request a new code (Step 1) |
| Skipped | Step 4 (`POST /v3/connect/account/`) fails — always verify OTP before creating the account |

---

## Step 4 — Create guest account

**Your backend** creates the account. Call this only **after Step 3 succeeds**.

### Guest handle (recommended)

**Omit `alias`** to get an auto-assigned guest handle (`guest000001`, `guest000002`, …):

```http
POST https://cloud.handcash.io/v3/connect/account/
Content-Type: application/json
app-id: YOUR_APP_ID
app-secret: YOUR_APP_SECRET

{
  "email": "user@example.com",
  "accessPublicKey": "<accessPublicKey hex from Step 2>"
}
```

### Custom handle (optional)

You **can** include `alias` to set a specific handle at creation time:

```json
{
  "email": "user@example.com",
  "accessPublicKey": "<accessPublicKey hex from Step 2>",
  "alias": "myuser"
}
```

This sets a **custom handle** at creation time while keeping the same app-created guest behavior as auto-assigned handles. The account is marked `guest: true`, uses your chosen alias instead of `guest0XXXXX`, and shares the same Connect spend cap and re-auth flow. Omit `alias` only when you want HandCash to auto-assign a `guest0XXXXX` handle.

| | Omit `alias` | Include `alias` |
|--|--------------|-----------------|
| Handle | Auto-assigned `guest0XXXXX` | Your chosen alias |
| `guest` flag | `true` | `true` |
| Connect spend cap | ~$50 USD | ~$50 USD |
| Re-request email code | Yes | Yes |
| HandCash app login | After [handle claim](#upgrade--claim-a-permanent-handle) | After [handle claim](#upgrade--claim-a-permanent-handle) |

**Success response (200):**

```json
{
  "id": "507f1f77bcf86cd799439011",
  "handle": "guest000042",
  "paymail": "guest000042@<paymail-domain>",
  "displayName": null,
  "avatarUrl": null,
  "localCurrencyCode": "USD",
  "bitcoinUnit": "BSV",
  "createdAt": "2026-06-23T12:00:00.000Z"
}
```

Your app is **automatically authorized** for this user. Persist `authToken` (private key hex) encrypted, mapped to your internal user ID.

### Returning guest users

If the same email already has a **guest** account, cloud re-authorizes your app and returns the existing guest handle. No duplicate wallet is created.

### Guest account limits

Applies to all Connect app-created accounts (with or without a custom `alias`):

| Limit | Value |
|-------|-------|
| Connect spend cap | ~$50 USD equivalent (enforced on `Connect.pay`) |
| Handle | Auto-assigned `guest0XXXXX` when `alias` is omitted; your chosen alias when provided |
| HandCash app login | Not available until handle is [claimed](#upgrade--claim-a-permanent-handle) |

---

## Step 5 — Operate the wallet with the SDK

All wallet operations use `@handcash/sdk` on your **server** with the stored `authToken`.

```typescript
import { getInstance, Connect } from '@handcash/sdk';

const sdk = getInstance({
  appId: process.env.HANDCASH_APP_ID!,
  appSecret: process.env.HANDCASH_APP_SECRET!,
  baseUrl: 'https://cloud.handcash.io',
});

const client = sdk.getAccountClient(authToken);
```

### Get profile

```typescript
const { data: profile, error } = await Connect.getCurrentUserProfile({ client });
if (error) throw new Error(error.message);
console.log(profile?.handle); // guest000042
```

### Get spendable balance

Use **spendable** balances for guest wallets — they reflect the Connect spend limit.

```typescript
const { data: balances, error } = await Connect.getSpendableBalances({ client });
if (error) throw new Error(error.message);
```

### Send payment

```typescript
const { data: payment, error } = await Connect.pay({
  client,
  body: {
    instrumentCurrencyCode: 'BSV',
    denominationCurrencyCode: 'USD',
    description: 'In-app purchase',
    receivers: [{ destination: 'merchant-handle', sendAmount: 0.50 }],
  },
});

if (error) throw new Error(error.message);
console.log(payment?.transactionId);
```

### SDK methods available after guest creation

Same as standard Connect authorization:

- `Connect.getCurrentUserProfile`
- `Connect.getSpendableBalances` / `Connect.getBalances`
- `Connect.pay`
- `Connect.getItemsInventory`
- `Items.*` (transfer, lock, etc. per your app permissions)
- `Connect.getPublicUserProfiles` (app-scoped client, no user token)

All methods return `{ data, error }` — always check `error` before using `data`.

---

## Detecting guest vs full accounts

Guest and full accounts share the same Connect profile shape. **Do not** rely on the `guest0` handle prefix — custom-alias guests use a normal-looking handle but are still app-created until claim.

### Recommended — track claim state in your app

Store whether the user has completed HandCash claim (e.g. after they return from the claim URL). Until then, treat them as a guest for upgrade CTAs and spend-limit messaging.

### Handle prefix (auto-assigned guests only)

Auto-assigned guests use `guest0XXXXX`. This check only covers that subset:

```typescript
const isAutoAssignedGuest = profile?.handle?.startsWith('guest0') ?? false;
```

Custom-alias guests will return `false` here even though they are still app-created.

### HandCash auth / account preview

HandCash auth state and account preview expose `isAppCreatedAccount: true` when the wallet was created via Connect and **has not yet been claimed** (`guestCreatedAt` set, `guestConvertedAt` not set). This covers both auto-assigned and custom-alias guests.

**UI guidance:**

- Show the user's handle clearly (whether `guest000042` or a custom alias)
- Prompt upgrade before spend limit or when the user needs full HandCash app access

---

## Upgrade — claim a permanent handle

When a guest user wants a real HandCash identity, redirect them to **HandCash web** to claim a username. This is **not** a Connect SDK call.

### What happens on claim

- **Same wallet** — balance, items, and history are preserved
- Handle is confirmed or changed at claim (auto-assigned `guest0XXXXX` → user-chosen alias; custom alias can be kept)
- User can log into HandCash mobile/web
- Your Connect authorization **remains valid** with the original `authToken` (unless the user revokes your app)

### What your app should do

1. Show CTA: *"Claim your HandCash username"*
2. Redirect to HandCash claim URL (same email as guest creation)
3. Pass a `returnTo` URL so the user returns to your app after claim
4. On return, refresh profile via `Connect.getCurrentUserProfile` — after claim, the user is a full HandCash account (same handle or updated)

**Claim URL (HandCash web):**

```
https://handcash.io/my-account/account/change-username?returnTo=<url-encoded-your-app-url>
```

Optionally append your app ID for partner branding (when supported):

```
https://handcash.io/my-account/account/change-username?appId=YOUR_APP_ID&returnTo=<url-encoded-your-app-url>
```

The user must sign in with **the same email** used during guest creation.

### After claim — refresh in your app

```typescript
const { data: profile } = await Connect.getCurrentUserProfile({ client });
// profile.handle is now the permanent alias, e.g. "satoshi"
```

---

## Existing HandCash users (Connect auth)

If the user **already has a full HandCash account**, guest creation is blocked at Step 1. Use standard Connect authorization instead.

### Generate redirect URL

```typescript
import { getInstance } from '@handcash/sdk';

const sdk = getInstance({
  appId: process.env.HANDCASH_APP_ID!,
  appSecret: process.env.HANDCASH_APP_SECRET!,
});

const url = sdk.getRedirectionUrl({ state: 'your-csrf-token' });
// https://app.handcash.io/#/authorizeApp?appId=...&state=...
```

Redirect the user to this URL. After authorization, HandCash redirects to your **Authorization Success URL** with `authToken` in the query string.

### Handle callback (example)

```typescript
// GET /auth/handcash/success?authToken=...&state=...
const authToken = req.query.authToken as string;
const state = req.query.state as string;

// Validate CSRF state, then store authToken securely
const client = sdk.getAccountClient(authToken);
const { data: profile } = await Connect.getCurrentUserProfile({ client });
```

---

## Error handling

| Scenario | Response | Your action |
|----------|----------|-------------|
| Feature not enabled | 403 `createAccounts` | Contact HandCash |
| Full account exists | 400 on requestEmailCode | Use Connect auth redirect |
| Invalid OTP | 4xx from Trustholder `/auth/verifyCode` | Let user retry; re-request code if expired |
| OTP verify skipped | 400 on create account | Call Step 3 before Step 4 |
| Invalid accessPublicKey | 400 | Regenerate keypair and restart from Step 2 |
| Spend limit exceeded | Connect pay error | Prompt handle claim |
| Guest returns (same email) | 200, same handle | Reuse stored authToken or issue new authorization |
| User revokes app | SDK auth errors | Re-run Connect auth redirect |
| Claim attempted via signUp | 409 `account already exists` | Direct user to HandCash claim URL instead |

---

## Security requirements

- **Never** expose `app-secret` in client-side code, mobile apps, or public repos
- **Encrypt** `authToken` at rest; treat it like a password
- **Never** log private keys or full auth tokens
- Use **HTTPS** for all redirects and API traffic
- Use **CSRF `state`** parameter on Connect auth redirects
- **Rate-limit** guest creation on your backend (per IP and per email)
- Validate **`returnTo`** URLs on redirect (allowlist your domains only)

---

## Reference implementation

### Backend service (Node.js / TypeScript)

```typescript
import { getInstance, Connect } from '@handcash/sdk';

const CLOUD_BASE = 'https://cloud.handcash.io';
const TRUSTHOLDER_BASE = 'https://trust.hastearcade.com';
const APP_ID = process.env.HANDCASH_APP_ID!;
const APP_SECRET = process.env.HANDCASH_APP_SECRET!;

async function cloudPost<T>(path: string, body: object): Promise<T> {
  const res = await fetch(`${CLOUD_BASE}${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'app-id': APP_ID,
      'app-secret': APP_SECRET,
    },
    body: JSON.stringify(body),
  });

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`HandCash API ${res.status}: ${text}`);
  }

  return res.json() as Promise<T>;
}

/** Step 1 */
export async function requestGuestEmailCode(email: string) {
  return cloudPost<{ requestId: string }>(
    '/v3/connect/account/requestEmailCode',
    { email },
  );
}

/** Step 3 */
export async function verifyGuestEmailOtp(params: {
  requestId: string;
  verificationCode: string;
  accessPublicKey: string;
}) {
  const res = await fetch(`${TRUSTHOLDER_BASE}/auth/verifyCode`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      requestId: params.requestId,
      verificationCode: params.verificationCode,
      publicKey: params.accessPublicKey,
    }),
  });

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`Trustholder API ${res.status}: ${text}`);
  }
}

/** Step 4 — omit alias intentionally for guest handle */
export async function createGuestWallet(email: string, accessPublicKey: string) {
  return cloudPost<{
    handle: string;
    paymail: string;
    id: string;
  }>('/v3/connect/account/', { email, accessPublicKey });
}

/** Steps 3 + 4 — verify OTP then create (recommended single backend call) */
export async function verifyAndCreateGuestWallet(params: {
  email: string;
  requestId: string;
  verificationCode: string;
  accessPublicKey: string;
}) {
  await verifyGuestEmailOtp({
    requestId: params.requestId,
    verificationCode: params.verificationCode,
    accessPublicKey: params.accessPublicKey,
  });
  return createGuestWallet(params.email, params.accessPublicKey);
}

/** Step 5 */
export function getConnectClient(authToken: string) {
  return getInstance({ appId: APP_ID, appSecret: APP_SECRET }).getAccountClient(authToken);
}

export async function getGuestProfile(authToken: string) {
  const client = getConnectClient(authToken);
  const { data, error } = await Connect.getCurrentUserProfile({ client });
  if (error) throw new Error(error.message);
  return data;
}

/** True only for auto-assigned guest0XXXX handles — not custom-alias guests */
export function isAutoAssignedGuestHandle(handle: string | undefined): boolean {
  return !!handle?.startsWith('guest0');
}

export function getClaimHandleUrl(returnTo: string, appId?: string): string {
  const params = new URLSearchParams({ returnTo });
  if (appId) params.set('appId', appId);
  return `https://handcash.io/my-account/account/change-username?${params.toString()}`;
}
```

### Suggested API routes in your app

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/handcash/guest/request-code` | POST | Body: `{ email }` → calls Step 1, returns `requestId` |
| `/api/handcash/guest/create` | POST | Body: `{ email, requestId, verificationCode, accessPublicKey, authToken }` → Steps 3 + 4, stores `authToken` |
| `/api/handcash/wallet/profile` | GET | Uses stored authToken → SDK profile |
| `/api/handcash/wallet/pay` | POST | Server-side payment via SDK |

The client collects the OTP, generates the keypair (Step 2), then sends everything to your create endpoint. Your backend verifies the OTP (Step 3) before creating the account (Step 4).

---

## Testing checklist

Use sandbox / staging credentials from HandCash where available.

- [ ] `createAccounts` feature enabled on your App ID
- [ ] Request email code returns `requestId`
- [ ] OTP verify succeeds via `/auth/verifyCode`
- [ ] Guest account created with `guest0XXXXX` handle (no alias in request)
- [ ] `Connect.getCurrentUserProfile` returns guest handle
- [ ] `Connect.getSpendableBalances` returns balances
- [ ] `Connect.pay` succeeds within spend limit
- [ ] Pay fails gracefully above spend limit with clear user message
- [ ] Same email + guest re-creates authorization without duplicate wallet
- [ ] Full-account email returns 400 on requestEmailCode → Connect auth works instead
- [ ] Claim redirect opens HandCash; after claim, profile handle updates
- [ ] Connect SDK still works with original authToken after claim
- [ ] `app-secret` never appears in client network tab

---

## FAQ

**Can I create guest wallets from `@handcash/sdk` alone?**  
No. Account creation uses Connect REST endpoints. The SDK is for operations after you have `authToken`.

**Can I set a custom handle at creation?**  
Yes — pass optional `alias` in Step 4 to set the handle upfront. The account is still created as an app-created guest (`guest: true`) with the same spend cap and re-auth behavior as auto-assigned `guest0XXXXX` handles. Omit `alias` only when you want HandCash to assign the handle for you.

**How do I submit the OTP?**  
`POST /v3/connect/account/` does not accept the verification code. Submit it separately to Trustholder: `POST https://trust.hastearcade.com/auth/verifyCode` with `{ requestId, verificationCode, publicKey }`. See [Step 3](#step-3--submit-email-otp). Call this before create account.

**Do I call Trustholder directly?**  
Yes, for OTP verification only (Step 3). Account creation and wallet operations use `cloud.handcash.io`. No `app-secret` is needed for the Trustholder verify call.

**Does the user need the HandCash app to use a guest wallet?**  
No. Guest wallets work entirely inside your app via Connect SDK until they choose to claim a handle.

**What happens to my app's authorization after claim?**  
It stays active. The same `authToken` continues to work unless the user revokes your app in HandCash settings.

**Can I use `@handcash/handcash-connect` (v2)?**  
v2 is deprecated. Use `@handcash/sdk` for new integrations. Neither v2 nor v3 SDK includes guest account creation.

**What if I call `/v3/account/signUp` for a guest email?**  
It fails with `409 account already exists`. Guests must use the HandCash claim flow (`/v3/account/complete/noPhone` on HandCash side).

---

## Support

| Need | Contact |
|------|---------|
| Enable `createAccounts` on your App ID | [Support@handcash.io](mailto:Support@handcash.io) |
| Dashboard / credentials | [dashboard.handcash.io](https://dashboard.handcash.io) |
| Community | [discord.handcash.io](https://discord.handcash.io) |

---

## Quick reference

```
CREATE GUEST
  1. POST /v3/connect/account/requestEmailCode     (backend, app-secret) → requestId
  2. User enters OTP from email                     (client UI)
  3. Generate keypair                               (client)
  4. POST trust.hastearcade.com/auth/verifyCode
       { requestId, verificationCode, publicKey }  (client or backend)
  5. POST /v3/connect/account/  { email, accessPublicKey [, alias] }  (backend; omit alias for guest)

USE WALLET
  sdk.getAccountClient(authToken) → Connect.pay / getSpendableBalances / …

DETECT GUEST (in your app)
  Track claim completion locally, or use isAutoAssignedGuestHandle(handle) for guest0 only
  HandCash account preview: isAppCreatedAccount (both auto + custom alias, until claim)

UPGRADE
  Redirect → https://handcash.io/my-account/account/change-username?returnTo=...

EXISTING HC USER
  sdk.getRedirectionUrl() → authorizeApp → authToken callback
```
