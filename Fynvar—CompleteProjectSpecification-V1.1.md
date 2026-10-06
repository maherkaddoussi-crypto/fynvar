# Fynvar — Complete Project Specification (v1.1, Consolidated)

> **Document status:** Consolidated specification v1.1. It merges the original v1.0 project specification with the expert review and revision delta.
> **Precedence rule:** Every v1.1 correction has been applied in place. Where any older wording conflicts with a v1.1 rule, the v1.1 rule wins. Appendix A lists every change and its reason.

---

## Change Tags Used in This Document

Tags mark content that was added or changed in v1.1:

| Tag                 | Meaning                                                              |
| ------------------- | -------------------------------------------------------------------- |
| **[FIX]**     | The v1.0 spec would have failed, lost money, or contradicted itself. |
| **[HARDEN]**  | Security or integrity improvement.                                   |
| **[PREMIUM]** | Polish at the level of Stripe or Adyen.                              |
| **[METAX]**   | Fitting Fynvar to the MetaX.Academy estate.                          |

Content without a tag is carried over unchanged from v1.0.

---

## Table of Contents

- **Part 1 — Project Description**
  - 1.1 Executive Summary
  - 1.2 Brand and Naming
  - 1.3 Hosting, Domains and Topology
  - 1.4 Technology Stack
  - 1.5 Data Model Principles
  - 1.6 The Ledger (Overview)
  - 1.7 Currency, Peg and Backing
  - 1.8 Feature Modules
  - 1.9 Security Posture
  - 1.10 Compliance Considerations
  - 1.11 MetaX.Academy Integration
  - 1.12 Design System
  - 1.13 File Layout (Target)
  - 1.14 Build Phases
  - 1.15 Out of Scope for v1
- **Part 2 — Technical Architecture**
  - 2.1 Ledger: Sign Convention, Chart of Accounts, Journals, Invariants
  - 2.2 Tamper Evidence
  - 2.3 Concurrency Pattern
  - 2.4 Binance Pay Integration
  - 2.5 Identity, Sessions and Access
  - 2.6 Idempotency
  - 2.7 API Conventions
  - 2.8 Merchants, Checkout, Subscriptions and Refunds
  - 2.9 Settings Architecture
  - 2.10 Testing Strategy
- **Part 3 — Admin Configuration Catalog (21 groups)**
- **Part 4 — Super AI Agent Prompt**
- **Appendix A — v1.0 → v1.1 Change Log**
- **Appendix B — Top Risks and Highest-Value Additions**

---

## Part 1 — Project Description

### 1.1 Executive Summary

**Fynvar** is a closed-loop digital payment platform. It issues its own internal currency, the **Finvar (ticker: FIN)**, which users fund with USDT deposits through Binance Pay.

The architecture follows a Stripe-like payment processor:

- Users hold a balance.
- Users transfer instantly between handles.
- Users deposit from and withdraw to external rails.
- Third-party merchants accept FIN for products and subscriptions.

The difference from Stripe is the unit of account. Fynvar does not use fiat. It uses FIN, which is pegged to USDT at a fixed rate and backed 1:1 by reserves held in a treasury account.

Fynvar is not a crypto token, and there is no blockchain. FIN exists only as entries in a double-entry ledger. It is a **programmable balance**, like airline miles, store credit or Stripe's own balance, with a full payments API on top.

### 1.2 Brand and Naming

- **Company / brand / domain:** Fynvar
- **Currency unit (singular/plural):** Finvar
- **Ticker:** FIN
- **Symbol:** ₣ (decorative only). **[FIX]** ₣ is the historical French franc sign. Use "FIN" as the main UI label and ₣ only as decoration.
- **API path:** `/fynvar-api/`. **[FIX]** v1.0 also used the spelling `fynval-api`. Keep `/fynval-api/` only as a 308 redirect alias. The canonical spellings are: brand = **Fynvar**, unit = **Finvar**, path = **`fynvar-api`**.
- **Motto:** "A balance built for software, not banks."
- **Voice:** Precise, calm, technical and confident. Stripe's clarity with a warmer, more human tone.

### 1.3 Hosting, Domains and Topology

**[FIX] Same-site topology is mandatory.**

In v1.0, the API sat on `gloulou.net` and the frontend on `pay.metax.academy`, with the refresh token in a `SameSite=None` cookie. That design breaks for two reasons:

- These are different sites. Safari ITP, Firefox and Chrome increasingly block third-party cookies, so token refresh fails for many users.
- Double-submit CSRF cannot work, because JavaScript on `pay.metax.academy` cannot read a cookie set by `gloulou.net`.

**v1 topology (choose one):**

| Option                    | Public API URL                                         | Backend                     |
| ------------------------- | ------------------------------------------------------ | --------------------------- |
| **A (recommended)** | `pay.metax.academy/api/*` (reverse-proxied)          | `gloulou.net/fynvar-api/` |
| B                         | `api.pay.metax.academy` (DNS points at the API host) | `gloulou.net/fynvar-api/` |

- **Primary static host (v1):** `pay.metax.academy`
- **API backend host (v1):** `gloulou.net/fynvar-api/`. v1.0 mentioned both `gloulou.com` and `gloulou.net`. This spec assumes `gloulou.net`, and the developer must confirm it.
- **Future primary domain:** `fynvar.com`, to be purchased and migrated to.
- **Subdomains after migration:** `fynvar.com` for marketing, `app.fynvar.com` for the dashboard and `api.fynvar.com` for the API. These are all the same site.
- **All static pages** are HTML + JavaScript + Tailwind CSS, with no server-side rendering.
- **All data operations** are HTTP calls to the PHP API.

### 1.4 Technology Stack

| Layer                 | Technology                                       | Notes                                                                                                                             |
| --------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| Static pages          | HTML5, vanilla JS (ES2022 modules), Tailwind CSS | Hosted on `pay.metax.academy`. **[HARDEN]** Build Tailwind ahead of time; do not use the CDN, so the CSP can stay strict. |
| API                   | PHP 8.2+                                         | Backend at `gloulou.net/fynvar-api/`, exposed same-site                                                                         |
| Database              | SQLite 3, WAL mode,`STRICT` tables             | Single file, encrypted at rest via the host volume. Must be portable to PostgreSQL through a PDO driver swap.                     |
| Crypto library        | libsodium via PHP `sodium_*`                   | Secret encryption, Ed25519 token signing                                                                                          |
| Password hashing      | Argon2id via `password_hash()`                 | Never bcrypt. The effective parameters are detected at boot.                                                                      |
| External payment rail | Binance Pay Merchant API v2                      | HMAC-SHA512 request signing, RSA-SHA256 webhook verification, certificates from the Query Certificate API                         |
| Web server            | Nginx or Apache with PHP-FPM                     | HTTPS enforced, HSTS, reverse proxy for `/api/`                                                                                 |
| Static CDN            | Cloudflare (or equivalent)                       | For `pay.metax.academy`                                                                                                         |
| Email                 | SMTP (Postmark / SES / Mailgun)                  | Transactional emails                                                                                                              |
| Monitoring            | Uptime check + error log webhook                 | Alerts the admin on failure                                                                                                       |

### 1.5 Data Model Principles

Every table follows these rules:

1. **Integer primary key** (`INTEGER PRIMARY KEY AUTOINCREMENT`) for internal joins and performance.
2. **Public ID column** for every external identifier: API responses, URLs and webhook payloads.
   - **[PREMIUM]** Public IDs are prefixed and time-ordered: `usr_`, `acct_`, `tx_`, `wd_`, `dep_`, `prod_`, `cs_`, `sub_`, `whd_`, followed by a **UUIDv7**. These replace `lower(hex(randomblob(16)))`.
   - UUIDv7 is time-ordered and index-friendly.
3. **Timestamps as TEXT** in true ISO-8601 UTC.
   - **[FIX]** v1.0 said `datetime('now')` produces ISO-8601. It doesn't: it returns `YYYY-MM-DD HH:MM:SS`, with no `T`, no `Z` and no milliseconds.
   - Use `strftime('%Y-%m-%dT%H:%M:%fZ','now')` everywhere.
4. **JSON metadata columns** (TEXT) where a flexible schema is needed, such as transactions and the audit log. Query them with `json_extract()`.
5. **No floats for money.** All amounts are integers in minor units.
6. **[HARDEN] Strict typing.** All tables are `STRICT`. Money columns carry `CHECK (typeof(amount) = 'integer')`.
7. **[PREMIUM] `livemode` flag** on every object that can exist in test mode. Test mode also uses a separate SQLite file (see §2.7).

### 1.6 The Ledger (Overview)

The ledger is the heart of Fynvar. Everything else is plumbing. Part 2 contains the full specification.

- **Double-entry.** Every money movement is a transaction with two or more entries that sum to zero per currency.
- **Per-currency balance.** A single transaction can move USDT and FIN together, as deposits and withdrawals do. Each currency's entries balance independently.
- **Sign convention [FIX].** `amount > 0` raises the account's balance. System accounts may go negative. A negative balance on `mint_*` *is* the outstanding liability.
- **Immutability.** Ledger entries are never updated or deleted. Corrections are posted as reversing entries.
  - **[HARDEN]** Database triggers enforce this.
  - **[HARDEN]** A SHA-256 hash chain makes tampering detectable.
- **Derived balances.** The `account_balances` table is a materialized cache. The source of truth is the sum of `ledger_entries.amount` per account and currency. Reconciliation checks that the two match.
- **System accounts.** Treasury, mint, withdrawal hold, payout clearing, fee revenue, processor fees and adjustments. They have no user and use a `code` such as `system:treasury`.
- **Idempotency.** Every mutating transaction carries an idempotency key. Replays return the original result without side effects. **[FIX]** Keys are scoped and fingerprinted (§2.6).
- **Minor units.** FIN has 2 decimals and USDT has 6, both stored as integers.
- **Concurrency [FIX].** SQLite has no row locks and no `SELECT … FOR UPDATE`. Every money transaction runs inside `BEGIN IMMEDIATE`. Debits are guarded updates that fail when funds are insufficient (§2.3).

### 1.7 Currency, Peg and Backing

- **Peg:** 1 USDT = 100 FIN, so 1 FIN = $0.01.
  - Because FIN has 2 decimals and USDT has 6, **1 FIN minor unit = 100 USDT minor units**.
  - Conversion is exact: `usdt_minor = fin_minor × 100`, with no rounding.
- **Peg mode [FIX]:** `fixed` (default) or `admin`. The `dynamic` mode is **removed**.
  - A floating rate can't coexist with "every FIN is redeemable 1:1 for USDT".
  - Changing the peg revalues every existing balance after the fact.
- **Immutability [FIX]:** After the first ledger entry, the peg, FIN decimals and USDT decimals become **immutable**. They can only change through a documented migration with a full re-statement.
- **Backing:** Every FIN in circulation is matched by USDT held in the treasury account. The admin can set a minimum reserve ratio. If the ratio is breached, deposits and/or withdrawals pause automatically.
- **Processor fees [FIX]:** If Binance charges merchant fees, the treasury receives less USDT than the FIN minted, and the reserve ratio falls with every deposit. To prevent this:
  - Book processor fees to the `system:processor_fees` expense account.
  - Price the deposit fee to cover them, or have the operator top up reserves.
  - Reconciliation must detect the gap.
- **Referral rewards [FIX]:** Rewards are never minted, because minted rewards would create FIN with no USDT behind it. They are paid from `system:fee_revenue_fin`, which is already backed. A reward is rejected if that balance is too low.

### 1.8 Feature Modules

**Authentication.**

- Email + password signup.
- Handles (`@alice`) are unique, case-folded and must match `^[a-z0-9_]{3,20}$`. **[HARDEN]**
- Reserved handles cannot be registered: admin, support, fynvar, metax, binance, etc.
- A released handle stays quarantined for 90 days to prevent impersonation.
- Passwords are hashed with Argon2id.
- 2FA uses TOTP (RFC 6238) with replay protection and 10 hashed backup codes.
  - **Passkeys (WebAuthn) are mandatory for admins.**
  - SMS is not offered (SIM-swap risk).
- Email verification can be enabled.
- Sessions use a short-lived access token (kept in memory) and a rotating refresh token in a `__Host-` cookie with `SameSite=Strict`, with reuse detection.
- Login is rate-limited, with lockout.

**Wallet.**

- The dashboard shows FIN balance, USDT balance (if any), handle and recent activity.
- Balances update in near real time through polling (default 15 s, configurable). Websockets are a future option.
- Balance changes animate with a count-up tween. The tween is skipped under `prefers-reduced-motion`.
- Amounts arrive as integer strings and are handled with `BigInt`.

**Deposits.**

1. The user selects an amount: minimum 100 USDT, step 100 USDT by default (admin-configurable).
2. The backend checks that the deposit would not push the user past `max_balance`.
3. The backend creates a Binance Pay order and returns a checkout URL, QR code link and deep link.
4. The user pays on Binance.
5. The webhook arrives at the API's `/webhooks/binance` endpoint and is verified with RSA-SHA256, using a certificate fetched from Binance and cached by serial number.
6. **[FIX]** Before crediting, the backend **re-queries the order** through Binance's Query Order API. It credits only if the status is success, the currency is `USDT` and the paid amount **equals the intended amount exactly**.
7. FIN is credited at 100 FIN per USDT. Processor fees are booked separately.
8. The credit is idempotent on `binance:pay:<merchantTradeNo>`.

**Withdrawals [FIX].**

1. The user specifies an amount (minimum 10,000 FIN by default) and a **payout destination**: a Binance `receiveType` (`BINANCE_ID` or `EMAIL`) plus a `receiver`.
   - Binance Pay Payout cannot send to on-chain addresses.
   - Destinations are validated with Binance's *Payout Validate Receiver* API before they are added to the allowlist.
   - New destinations have a cooldown.
2. 2FA is required, re-prompted at the moment of the action.
3. The withdrawal follows a **two-phase** flow:
   - **HOLD:** On request, FIN moves from the user to `system:withdrawal_hold`.
   - **Approval:** Auto, admin, or threshold-based.
   - **Payout:** Sent through Binance Batch Payout.
   - **SETTLE:** On confirmed payout success, the FIN is burned and the USDT leaves the treasury.
   - **RELEASE:** On rejection or terminal payout failure, the FIN returns to the user.
4. FIN is never burned before the payout succeeds.

**Transfers.**

- Instant peer-to-peer transfers between handles.
- Fee is admin-configurable, default 0.25%.
- The sender's debit is guarded inside `BEGIN IMMEDIATE`. The fee is posted to `fee_revenue_fin` and the net amount to the recipient.
- An idempotency key is required.
- Self-transfers are not allowed.
- Daily and per-transaction limits are configurable.
- **[PREMIUM]** Before confirming, the user sees the recipient preview, gross amount, fee and net amount, with "Arrives instantly · Irreversible".

**Merchant Platform.**

- A user can register as a merchant. KYB runs when admin approval is enabled.
- Merchants receive API keys in test and live variants:
  - Format: `fyn_test_…` / `fyn_live_<8-char prefix>_<32 random>`.
  - Each key is shown once and stored as `HMAC-SHA256(pepper, key)`.
  - Scopes: `checkout:write`, `products:write`, `refunds:write`, `read_only`.
- Merchants create products with a name, description, price in FIN and an optional recurring interval.
- Merchants receive FIN for sales.
- Payout schedule: on-demand, daily, weekly or monthly.
- Payout currency: FIN (internal transfer), USDT (through the withdrawal pipeline) or both.
- **[PREMIUM]** A configurable rolling reserve protects against merchants who disappear.
- **[PREMIUM][METAX]** Split payments: `application_fee_bps` per checkout, following the Stripe Connect pattern.

**Checkout.**

1. A merchant's site calls the API with a product ID. The buyer handle is optional; requiring it would leak handles.
2. The API returns a `checkoutUrl` of the form `pay.metax.academy/checkout.html?c=<cs_id>`.
3. **[FIX]** The price is **snapshotted from the product** into the session. It is never taken from the merchant's request.
4. `success_url` and `cancel_url` must match the merchant's registered domains.
5. The buyer logs in, sees the product and confirms.
6. The ledger posts a transfer from buyer to merchant, minus the merchant fee (and the application fee, if any).
7. A signed webhook fires to the merchant.
8. A session can be paid only once.

**Subscriptions.**

- **[FIX]** A buyer authorizes a **mandate** covering the merchant, product, maximum amount per period and interval.
- A cron job (`/cron/charge-subscriptions`) charges due subscriptions with the idempotency key `sub:<id>:<period_start>`, so running the cron twice cannot double-charge.
- The cron never charges more than the mandate allows. A price increase requires new consent from the buyer.
- On success, the period advances.
- On failure, the subscription moves to `past_due` and a dunning sequence starts.
- A pre-renewal reminder is emailed.
- Merchant webhooks fire on every renewal, failure and cancellation.

**Admin Panel.**

- Lives at `pay.metax.academy/admin.html`.
- Gated by the `is_admin` flag and a **passkey**, with an optional IP allowlist and 15-minute sessions.
- Contains:
  - Tabs for all 21 configuration groups.
  - A user list (impersonation off by default).
  - A transaction browser.
  - A withdrawal approval queue.
  - A merchant list with KYB approval.
  - Fee change history, including scheduled changes.
  - A reserve dashboard.
  - A manual ledger adjustment tool. It posts balanced entries against `system:adjustments` only and requires a reason string.
- **[HARDEN]** **Maker–checker (four-eyes)** approval applies to manual adjustments, fee changes, peg-related settings and withdrawals above a threshold.

**Notifications.**

- Transactional emails for: signup, deposit confirmation, withdrawal held/approved/settled/released, transfer received, subscription reminder/renewal/failure, and security alerts (including refresh-token reuse).
- Templates are stored in the database and editable by the admin.
- An optional admin webhook covers large deposits, large withdrawals, failed-login spikes, reserve alerts and reconciliation mismatches.

**Audit Log.**

- Every admin action is recorded: fee change, config change, user suspension, manual adjustment, withdrawal approval and maker–checker decisions.
- Each record holds the actor ID, timestamp, old value, new value and reason. For dual-approved actions, both proposer and approver are recorded.

**Transparency [METAX].**

- A public, falsifiable `/reserves` page.
- A public correction log.
- Declared-limits sections on the marketing and rates pages.

See §1.11 for details.

### 1.9 Security Posture

- HTTPS everywhere, HSTS and no mixed content.
- **[FIX]** Same-site API. Refresh cookie: `__Host-fyn_rt; Secure; HttpOnly; SameSite=Strict; Path=/`.
- **[FIX] CSRF:**
  - Bearer tokens in headers make the API endpoints CSRF-proof.
  - Only `/auth/refresh` and `/auth/logout` rely on cookies. They are protected by `SameSite=Strict`, a strict `Origin` allowlist and a required custom header `X-Fynvar-Client: web`.
- CORS is restricted to configured origins (`pay.metax.academy`; `fynvar.com` origins after migration), with credentials allowed.
- **Secrets:**
  - The root encryption key, Binance keys, the API-key pepper, SMTP credentials and S3 credentials live in environment variables, never in the repo.
  - If secrets are stored in the database, they are encrypted with a key-ID prefix (`k1:` + nonce + ciphertext) so keys can be rotated.
- Sensitive DB columns (TOTP secrets, merchant webhook secrets, any stored provider secrets) are encrypted with libsodium `secretbox`.
- **Passwords:** Argon2id (memory 64 MB, time 3, parallelism 4 as configured).
  - **[HARDEN]** Detect at boot whether PHP's Argon2 comes from libsodium or libargon2. Under libsodium, threads is effectively 1. Log the effective parameters.
  - Keep `max concurrent hashes × 64 MB` below the PHP-FPM memory budget, and tie auth rate limits to that bound.
- Merchant API keys are stored as peppered HMAC-SHA256 values. The plaintext is shown exactly once.
- Rate limiting per IP and per user applies to all auth and money endpoints, plus `/transfers/lookup` to slow handle enumeration.
- Payout destinations use an allowlist with cooldown and Binance receiver validation.
- 2FA is re-requested before money-moving actions: changing a payout destination, withdrawing and creating API keys.
- **[HARDEN] SSRF protection** for merchant webhooks:
  - HTTPS only.
  - DNS is resolved at send time; private, loopback and link-local ranges are refused (RFC 1918, 169.254/16, ::1, fc00::/7).
  - No redirects are followed.
  - Responses are capped at 4 KB.
- **[HARDEN] Strict CSP** on every payment page:
  - `default-src 'self'; script-src 'self'` (with hashes).
  - `connect-src` limited to the API.
  - `frame-ancestors 'none'` on checkout.
- **[HARDEN] Origin isolation.** No ad or analytics scripts on `pay.metax.academy`, and no cookies on `.metax.academy`.
- **Integrity checks:**
  - Balance integrity in CI: the sum of all FIN entries is zero and the sum of all USDT entries is zero.
  - Reconciliation (hourly or daily): `account_balances` equals aggregated `ledger_entries`, and the treasury USDT equals the custody balance (± in-flight clearing).
  - The hash chain is re-walked.
- A documented break-glass admin account is kept offline.

### 1.10 Compliance Considerations

> This section lists topics for legal counsel. It is not legal advice.

Fynvar holds user USDT and issues a redeemable internal balance. It has the features most regulators look for when deciding whether something is **electronic money**:

- prefunded value,
- redeemable at par,
- transferable between users,
- accepted by third-party merchants.

**Before the first real deposit, get a legal opinion on:**

- Whether Fynvar is an MSB, VASP or e-money issuer (or none of these) in the operating jurisdiction.
- **EU/EEA:** The E-Money Directive may apply. So may MiCA, as an e-money token, because the value is pegged to a single fiat currency through USDT. The "limited network" exemption usually stops applying once independent merchants join.
- **US:** Likely money transmission (FinCEN MSB registration plus state licences).
- **Everywhere:** FATF Recommendation 16 (the Travel Rule) may apply to the USDT legs.
- KYC/AML obligations and their thresholds.
- Consumer protection, refund and disclosure requirements, including effective-dated fee-change notice.

**Binance terms.** Binance Pay's merchant terms may not allow reselling or re-issuing the service as a stored-value platform. Get **written confirmation from Binance** before building Phase 5.

**Safeguarding.** Holding all reserves at a single exchange is concentration risk. Document how reserves are segregated, and write a **wind-down plan** describing how users get their USDT back if Fynvar shuts down.

**Disclosures.** Tell users plainly that FIN:

- is not a bank deposit,
- is not insured,
- depends on reserves held at Binance for redemption.

**Phase 0 gate [FIX].** A legal opinion, Binance's written approval of the use case, and published terms of service and refund policy are all required. **Phase 5 (deposits) cannot start until all three are recorded as complete in `DEPLOY.md`.**

The admin panel includes KYC/AML configuration (Group 12), so the operator can turn on identity verification, sanctions screening and transaction monitoring when required.

### 1.11 MetaX.Academy Integration [METAX]

MetaX's stated DNA is **Sidq** (truthfulness), **falsifiability**, **declared limits** and **correctability**, under Maher's License 2.3. Payments is one of the few industries where those principles produce something users can check.

**A falsifiable reserve page (`/reserves`, upgrading `security.html`).** It shows:

- FIN outstanding, treasury USDT and the reserve ratio, updated hourly.
- The daily ledger chain-head hash.
- The time of the last reconciliation and any mismatches found.
- A stated **falsifier**:

  > "This claim is false if treasury USDT / (FIN outstanding × 100) < 1.00 at any published snapshot, or if any published chain hash cannot be reproduced."
  >

This is proof of reserves expressed in MetaX's own idiom.

**Declared-limits section** on `index.html` and `rates.html`. It states what Fynvar is not:

- not a bank,
- not insured,
- not on-chain.

It also states the single-custodian setup, the SQLite single-writer ceiling, and which jurisdictions are unsupported.

**Public correction log.** Every incident, reconciliation mismatch and fee change gets an entry with its reason, including rejected bug reports. This mirrors MetaX's published correction queue.

**Use cases inside the estate:**

- MetaX.Academy is **merchant #1**, selling paid TopTech Academy tracks and certificates priced in FIN.
- **Academies Community** founders become merchants. Split payments give MetaX a platform fee on each academy's sales within one atomic journal.
- Subscriptions cover recurring curriculum access.

**Origin isolation [HARDEN].** MetaX.Academy runs third-party ads (AdSense, Adsterra), and third-party scripts on payment pages are the classic card-skimming vector. Therefore:

- No ad or analytics scripts on `pay.metax.academy`.
- Only host-only `__Host-` cookies; never set cookies on `.metax.academy`.
- Strict CSP on all payment pages.
- Tailwind is prebuilt.

**Licensing.** Decide whether Fynvar's client code is published under ML-2.3, and add a Part H statement for the platform itself. The economic harms in the 25-harm register, such as lock-in and opaque fees, map directly onto the disclosure features above.

### 1.12 Design System

**Stripe DNA:** generous whitespace, weight-300 large headlines, gradient hero surfaces, restrained micro-interactions, monospace accents for numbers, and a near-black canvas.

**Different colors:** a teal-to-violet primary gradient instead of Stripe's indigo, a warm amber accent for warnings, and a ruby accent for destructive actions.

**Palette:**

| Token         | Value                                      | Usage note                                                                                                                         |
| ------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Brand primary | teal `#06caae` (500), full 50–950 scale | **[PREMIUM]** Buttons use **ink `#07080c` text on teal** (about 9.6:1). White on teal is about 2.1:1 and fails WCAG. |
| Accent violet | `#7c5cff`                                | White on violet is about 4.3:1. Use it only for large text or bold labels of 18 px and up.                                         |
| Accent amber  | `#ffba27`                                | Warnings                                                                                                                           |
| Accent ruby   | `#ea2261`                                | Destructive actions, negative amounts                                                                                              |
| Ink canvas    | `#07080c`                                | Background                                                                                                                         |
| Panel         | `#101320`                                | Cards                                                                                                                              |
| Border        | `#1e2333`                                | Dividers                                                                                                                           |
| Muted text    | `#8b93ad`                                | About 6.5:1 on the canvas, which passes AA                                                                                         |

**Animations:**

- Gradient shift on hero backgrounds (400% background-size, 8 s loop).
- Fade-up on cards and modals (0.4 s cubic-bezier).
- Pulse-ring on the primary CTA during pending states.
- Count-up tween on balance changes (500 ms cubic ease-out).
- Skeleton shimmer while data loads.
- Subtle scale on button press (0.98).
- Slide-in toasts from the bottom right.
- **[PREMIUM] Reduced motion:** Respect `prefers-reduced-motion`. Disable the gradient shift, jump count-ups straight to the final value, and use fades only for toasts.

**Typography:**

- Inter (or the system UI font) for the interface.
- JetBrains Mono for handles, amounts, transaction IDs and API keys.
- **[PREMIUM]** Use `font-variant-numeric: tabular-nums` on every amount so count-ups don't jitter.
- Negative amounts use a true minus sign (−) in ruby. Incoming amounts use teal with a leading "+".

**Layout:** max-width 1080 px content column, sticky top nav with blur backdrop, 68 px nav height, 24 px horizontal padding on mobile.

**[PREMIUM] Money object states:**

- Designed visual states for `pending`, `held`, `settled`, `failed` and `reversed`.
- A timeline view on the transaction detail page.

**[PREMIUM] Receipts:**

- Downloadable HTML or PDF receipts with the transaction ID, entries, fee breakdown and chain hash.

**[PREMIUM] Trust micro-copy:**

- Transfers: show recipient preview, gross, fee and net, plus "Arrives instantly · Irreversible".
- Withdrawals: show the estimated settlement window and the destination's cooldown status.

### 1.13 File Layout (Target)

```
pay.metax.academy/
  index.html            → marketing landing (+ declared-limits section)
  dashboard.html        → wallet dashboard
  checkout.html         → hosted checkout (CSP frame-ancestors 'none')
  merchant.html         → merchant onboarding, keys, products, webhooks (+ verify snippets)
  subscriptions.html    → user's subscriptions / mandates
  admin.html            → admin panel (passkey-gated)
  login.html            → sign in / sign up
  rates.html            → public rates, current + scheduled fees, declared limits
  reserves.html         → falsifiable proof-of-reserves page (served at /reserves)
  corrections.html      → public correction log
  security.html         → security posture page (links to /reserves)
  receipt.html          → printable receipt view
  terms.html
  privacy.html
  assets/
    app.js              → shared client (API, auth, BigInt money formatting, toasts)
    crypto.js           → client helpers (idempotency key gen, WebAuthn helpers)
    styles.css          → prebuilt Tailwind (no CDN)
    tailwind.config.js
    animations.css

  /api/*  → reverse proxy → gloulou.net/fynvar-api/

gloulou.net/fynvar-api/          (/fynval-api/ → 308 alias)
  index.php             → front controller / router, Request-Id, error envelope
  config.php            → env loading, constants
  db.php                → PDO setup, WAL, busy timeout, BEGIN IMMEDIATE helper, BUSY retry
  money.php             → minor-unit conversion, bps math, rounding
  crypto.php            → libsodium encrypt/decrypt with key-ID prefix, Ed25519 tokens
  ids.php               → prefixed UUIDv7 generation
  auth.php              → sessions, token rotation/families, password, TOTP, backup codes
  webauthn.php          → passkey registration/assertion
  cors.php              → CORS + preflight + Origin allowlist
  csp.php               → security headers
  ratelimit.php         → per-IP and per-user limiter
  idempotency.php       → scoped keys, fingerprinting, stored responses
  ledger.php            → postJournal, hash chain, balance derivation
  accounts.php          → account lookup, system accounts, getOrCreate, guarded debit
  settings_registry.php → code-defined registry (types, defaults, validators, flags)
  settings.php          → override read/write with versioned cache
  approvals.php         → maker–checker workflow
  audit.php             → audit log writer
  binance.php           → Binance Pay client (order, query, cert, payout, validate receiver)
  webhooks/
    binance.php         → incoming webhook verifier (PAY + PAYOUT)
    merchant.php        → outgoing signed dispatcher with SSRF guard and retries
  routes/
    auth.php
    wallet.php
    deposits.php
    withdrawals.php
    destinations.php
    transfers.php
    rates.php
    reserves.php
    merchants.php
    apikeys.php
    products.php
    checkout.php
    subscriptions.php
    refunds.php
    admin.php
    cron.php
  cron/
    charge-subscriptions.php
    send-renewal-reminders.php
    poll-payouts.php
    reconcile.php
    anchor-chain-head.php
    retry-webhooks.php
    expire-pending.php
    apply-scheduled-fees.php
    release-rolling-reserve.php
    backup-db.php
  migrations/
    001_init.sql
    002_ledger_triggers.sql
    003_seed_system_accounts.sql
  tests/
  data/
    fynvar_live.sqlite
    fynvar_test.sqlite
    backups/
  logs/
    app.log
    audit.log
```

> **[FIX]** Settings defaults are no longer seeded as migrations. They live in `settings_registry.php`, and the database stores only overrides (§2.9).

### 1.14 Build Phases

| Phase       | Deliverable                                                                                                 | Exit Criteria                                                                                                                                            |
| ----------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **0** | Legal opinion, Binance written approval, ToS and refund policy, same-site domain topology                   | Signed off and recorded in `DEPLOY.md`. Same-site API reachable.                                                                                       |
| 1           | Migrations (STRICT, append-only triggers), settings registry, money helpers, ledger, hash chain, unit tests | Unbalanced journal rejected. Idempotency replay returns the same result. I1–I6 property tests pass. Append-only triggers proven.                        |
| 2           | Auth (token rotation, families, TOTP, backup codes, passkeys for admin) + wallet read endpoints             | Signup, login, refresh and `GET /wallet` return correct JSON. Refresh works in Safari.                                                                 |
| 3           | Internal FIN transfers                                                                                      | Two users can send FIN. Fee posted, balances correct. 50-process double-spend test passes.                                                               |
| 4           | Dashboard UI                                                                                                | Balance card, activity table, transfer modal with trust copy, toasts, animations, reduced motion.                                                        |
| 5           | Binance Pay deposits: signing, cert cache, webhook, re-query, processor fees                                | Order created, webhook verified, FIN credited, idempotent on replay. Replay and tamper tests pass. Reserve ratio unchanged after 1,000 sandbox deposits. |
| 6           | Withdrawals: destinations, receiver validation, hold, approval queue, payout, settle/release                | Allowlist and cooldown work. Every Binance error code maps to exactly one outcome.                                                                       |
| 7           | Merchant platform (test/live keys, scopes) + products + checkout + split payments                           | A merchant can create a product, an external site redirects to checkout, and payment settles.                                                            |
| 8           | Subscriptions (mandates) + refunds + cron                                                                   | Renewal, dunning, cancellation and webhooks work. MetaX.Academy is live as merchant #1 in sandbox.                                                       |
| 9           | Admin panel, all 21 config groups, maker–checker, effective-dated fees                                     | Every setting validated, audited, and dual-approved where flagged.                                                                                       |
| 10          | Hardening: rate limits, reconciliation,`/reserves`, correction log, backups with a restore drill          | A restore from backup reproduces the published chain head. Legal review signed off.                                                                      |

### 1.15 Out of Scope for v1

- Native mobile apps.
- Payment rails other than Binance Pay.
- Fiat on/off ramps.
- Non-USD stablecoin deposits.
- Referral payouts in USDT.
- A public API documentation site (verification snippets in `merchant.html` are in scope).
- Webhooks to users (merchant webhooks only).
- In-app chat or support ticketing.
- Dynamic or floating peg.
- On-chain payouts. Payouts go only to `BINANCE_ID` or `EMAIL` receivers.

---

## Part 2 — Technical Architecture

### 2.1 Ledger

#### 2.1.1 Sign convention (stated once, enforced everywhere) [FIX]

- `amount > 0` raises the account's balance.
- Each transaction's entries sum to zero per currency.
- System accounts may go negative. A negative balance on `mint_*` *is* the outstanding liability.
- Customer accounts may never go negative. The only exception is a monitored merchant negative balance backed by the rolling reserve (§2.8).

#### 2.1.2 Chart of accounts

| Code                                      | Currency | Meaning                                                  | Normal sign |
| ----------------------------------------- | -------- | -------------------------------------------------------- | ----------- |
| `system:treasury`                       | USDT     | USDT held at Binance (custody asset)                     | +           |
| `system:mint_usdt`                      | USDT     | USDT counter-entry for issued FIN                        | −          |
| `system:mint_fin`                       | FIN      | Issuance counter;`−balance` = FIN in circulation      | −          |
| `system:withdrawal_hold`                | FIN      | FIN reserved for withdrawals in flight                   | +           |
| `system:payout_clearing`                | USDT     | USDT sent but not yet confirmed                          | +           |
| `system:fee_revenue_fin`                | FIN      | Operator fee income (backed FIN); funds referral rewards | +           |
| `system:fee_revenue`                    | USDT     | Operator income kept in USDT                             | +           |
| `system:processor_fees`                 | USDT     | Binance fees (expense)                                   | +           |
| `system:merchant_reserve`               | FIN      | Rolling reserve held from merchant sales                 | +           |
| `system:adjustments`                    | both     | The only account manual adjustments may touch            | ±          |
| `user:<id>:fin` / `merchant:<id>:fin` | FIN      | Customer balances (liabilities)                          | +           |

> `system:merchant_reserve` implements the rolling reserve from §2.8. It is an implementation detail added in this consolidation.

#### 2.1.3 Journal catalog

**Rates:** FIN has 2 decimals, USDT has 6, and 1 USDT = 100 FIN. So **1 FIN minor = 100 USDT minor**. Conversion is exact: `usdt_minor = fin_minor × 100`.

```
DEPOSIT (verified X USDT minor, processor fee f USDT minor)
  treasury          USDT  +(X − f)
  processor_fees    USDT  +f
  mint_usdt         USDT  −X
  user:fin          FIN   +X/100        ; minted FIN
  mint_fin          FIN   −X/100
  ; If a deposit fee D (FIN minor) applies:
  ;   user:fin +(X/100 − D), fee_revenue_fin +D  (mint_fin still −X/100)

TRANSFER (gross G, fee F)
  sender:fin        FIN   −G
  recipient:fin     FIN   +(G − F)
  fee_revenue_fin   FIN   +F

CHECKOUT / SUBSCRIPTION CHARGE (gross G, platform fee F, application fee A, reserve R)
  buyer:fin         FIN   −G
  merchant:fin      FIN   +(G − F − A − R)
  fee_revenue_fin   FIN   +F
  platform_merchant:fin FIN +A          ; split payments only
  merchant_reserve  FIN   +R            ; rolling reserve only

REFUND
  ; reverse of the original charge shape (partial refunds pro-rata)

WITHDRAWAL.HOLD (on request: gross G, fee F)
  user:fin          FIN   −G
  withdrawal_hold   FIN   +G

WITHDRAWAL.SETTLE (payout webhook/query = SUCCESS; net N = G − F)
  withdrawal_hold   FIN   −G
  fee_revenue_fin   FIN   +F            ; fee stays as backed FIN
  mint_fin          FIN   +N            ; burn
  mint_usdt         USDT  +N×100
  treasury          USDT  −N×100        ; USDT left custody

WITHDRAWAL.RELEASE (rejected / terminal payout failure)
  withdrawal_hold   FIN   −G
  user:fin          FIN   +G

REFERRAL REWARD (amount R)
  fee_revenue_fin   FIN   −R            ; rejected if balance < R
  user:fin          FIN   +R
```

**Optional stricter in-flight view.** Route the USDT side of settlement through `payout_clearing`:

1. treasury → clearing when Binance answers `ACCEPTED`;
2. clearing → mint when the payout succeeds.

**Manual adjustments** always post balanced entries against `system:adjustments`. They require a reason and maker–checker approval.

#### 2.1.4 Invariants

These are checked in CI, at reconciliation and on the public reserve page.

```
I1  Σ FIN entries (all accounts) = 0           and   Σ USDT entries = 0
I2  account_balances[a,c] = Σ ledger_entries[a,c]   for all a, c
I3  FIN_outstanding = −balance(mint_fin)
I4  reserve_ratio = balance(treasury) / (FIN_outstanding × 100)   ≥ min_reserve_ratio
I5  balance(treasury) = Binance custody balance (± in-flight clearing)
I6  no customer account < 0
```

### 2.2 Tamper Evidence [HARDEN][METAX]

The ledger must be **falsifiable** as well as immutable.

- **Strict tables.** Use SQLite `STRICT` tables and `CHECK (typeof(amount) = 'integer')`.
- **Immutability triggers.** `BEFORE UPDATE` and `BEFORE DELETE` triggers on `ledger_entries` and `transactions` call `RAISE(ABORT, 'ledger is append-only')`.
- **Hash chain.** Every transaction stores `prev_hash` and `hash = SHA-256(prev_hash ‖ canonical_json(tx + entries))`. Reconciliation re-walks the chain.
- **Daily anchor.** Each day, the chain-head hash is published on the public reserve page. Anyone holding an older hash can detect history being rewritten.

### 2.3 Concurrency Pattern [FIX]

This replaces the v1.0 "row lock" wording. SQLite has no row locks and no `SELECT … FOR UPDATE`.

```
BEGIN IMMEDIATE                                  ; takes the database write lock
  check idempotency (scoped)
  guarded debit:
    UPDATE account_balances SET balance = balance - :amt
    WHERE account_id = :id AND currency = :cur AND balance >= :amt
    → 0 rows ⇒ InsufficientFunds (rollback)
  insert transaction + entries, upsert credits, compute hash chain
  verify Σ per currency = 0                     → else throw UnbalancedJournal
COMMIT
```

- Set `busy_timeout = 5000`, `synchronous = NORMAL` and `foreign_keys = ON`.
- Retry `SQLITE_BUSY` up to 3 times with jitter.
- **Declared limit:** SQLite allows one writer at a time. That is fine for v1 (hundreds of transactions per second).
- `postJournal` is written against PDO generically, so moving to PostgreSQL with `SELECT … FOR UPDATE` is a driver swap, not a rewrite.

### 2.4 Binance Pay Integration (aligned with current docs)

**Outgoing requests.**

- Headers:
  - `BinancePay-Timestamp` (milliseconds)
  - `BinancePay-Nonce` (32 random letters, a–z/A–Z)
  - `BinancePay-Certificate-SN` (the API key)
  - `BinancePay-Signature`
- Signature: `strtoupper(hash_hmac('sha512', ts."\n".nonce."\n".body."\n", secret))`.
- Sign the **exact** bytes you send. Encode the JSON once and reuse that string.

**Incoming webhook verification.**

1. Read the headers `BinancePay-Certificate-SN`, `-Nonce`, `-Timestamp` and `-Signature`.
2. Build the payload `timestamp + "\n" + nonce + "\n" + rawBody + "\n"` (LF = 0x0A).
3. Base64-decode the signature and verify it with `openssl_verify(..., OPENSSL_ALGO_SHA256)`.
4. Get the public key from the **Query Certificate API** (`certPublic`), looked up by Certificate-SN.
   - **[PREMIUM]** Cache certificates by SN and refresh when an unknown SN arrives. Binance can then rotate keys without an outage.
   - A manually pasted PEM is only a fallback.
5. **[FIX] Timestamp check.** Binance timestamps are in **milliseconds**. Normalize: `abs(now_ms − ts_ms) <= window_s × 1000`.
6. Reject replays: store `(cert_sn, nonce)` for the length of the replay window.
7. If the signature is invalid, return 401.

**Event handling.**

- Branch on `bizType = "PAY"` and `bizStatus = "PAY_SUCCESS"`. Also handle `PAY_CLOSED`.
- The `data` field is a **JSON string**, so decode it twice.
- Reply with the body `{"returnCode":"SUCCESS","returnMessage":null}`. An HTTP 200 alone is not enough.
- Look up the order by `merchantTradeNo`. It allows at most 32 alphanumeric characters, and a 32-character hex UUID fits exactly.
- If the order is not found (for example, the webhook arrived before the row was committed), schedule a re-check through Query Order rather than dropping the event.
- **[FIX] Before crediting, call Query Order** and compare status, currency (`USDT`) and amount (an **exact** match with the intended amount).
  - The webhook is a hint; the query is the proof.
  - On any mismatch, flag the order for manual review and do not credit.
- Credit with the idempotency key `binance:pay:<merchantTradeNo>` and book the processor fee.
- **`max_balance` overflow.** Enforce `max_balance` when the order is created. If a credit would still push the user past it, don't reject it, because Fynvar already holds the USDT. Credit it, freeze the excess and open a review.
- Deposits above the manual-review threshold are held for review before crediting.

**Order constraints.**

- `orderExpireTime`: the validation range is **5–60 minutes**. v1.0 allowed 5–1440, but Binance Pay is believed to cap it at 1 hour. Verify this against the Create Order docs for your API version.
- Required confirmations: removed. Binance Pay is off-chain and has no confirmations.

**Payouts.**

- Use **Batch Payout**.
  - `requestId` and `merchantSendId`: at most 32 characters, alphanumeric only.
  - `bizScene = "MERCHANT_PAYMENT"`, `transferMethod = "FUNDING_WALLET"`.
- Receivers: `receiveType` ∈ {`BINANCE_ID`, `EMAIL`} (plus `OPEN_ID` where applicable). Validate them first with **Payout Validate Receiver**.
- Binance caps each payout detail at about **80K USD equivalent**.
- Act on the payout webhook *and* poll **Payout Query**. SETTLE on success; RELEASE on failure.
- Error codes:

| Class               | Codes                                                                  | Action             |
| ------------------- | ---------------------------------------------------------------------- | ------------------ |
| **Terminal**  | `400703`, `400709`, `400710`, `400711`, `406103`, `406106` | RELEASE the hold   |
| **Retryable** | `400000`, `400501`                                                 | Retry with backoff |

- Every payout must end in exactly one SETTLE or one RELEASE, never both.

### 2.5 Identity, Sessions and Access

**Topology [FIX].** Same-site API (§1.3).

**Tokens.**

- Refresh cookie: `__Host-fyn_rt; Secure; HttpOnly; SameSite=Strict; Path=/`.
- Access token: an Ed25519-signed JWT (`sodium_crypto_sign`) or an opaque token. It lives in JavaScript memory and has a 15-minute TTL.
- Refresh tokens are **rotated on every use** and grouped into token families. If an old refresh token is reused, revoke the whole family and send a security alert.

**CSRF.**

- API endpoints authenticate with bearer tokens in headers, which makes them CSRF-proof.
- Only `/auth/refresh` and `/auth/logout` use cookies. They require `SameSite=Strict`, an allowlisted `Origin`, and the header `X-Fynvar-Client: web`.

**2FA [HARDEN].**

- TOTP (RFC 6238, ±1 step). Store the last used step to block code reuse.
- 10 single-use backup codes, hashed with Argon2id.
- **Passkeys (WebAuthn) are mandatory for admins**, because TOTP alone is phishable.
- SMS is not offered.
- 2FA is requested again before money-moving actions: changing a payout destination, withdrawing and creating an API key.

**Handles [HARDEN].**

- Must match `^[a-z0-9_]{3,20}$` and are case-folded.
- A reserved list blocks names such as admin, support, fynvar, metax and binance.
- Handle history is kept, and a released handle is quarantined for 90 days.
- `/transfers/lookup` is rate-limited and returns only display name and avatar.

**Admin hardening.**

- Optional IP allowlist.
- 15-minute admin sessions.
- **Maker–checker (four-eyes)** for manual ledger adjustments, fee changes, peg-related settings and withdrawals above a threshold. One admin proposes, a different admin approves, and both are audited.
- A documented break-glass account, kept offline.

### 2.6 Idempotency [FIX][PREMIUM]

- **Scope.** The key is `(principal_id, method, path, Idempotency-Key)`, where the principal is a user or merchant. Without scoping, two users who send the same key would collide.
- **Fingerprint.** A request hash is stored with each key. The same key with a different body returns **422 `idempotency_key_reused`**.
- **Stored response.** The full response (status and body) is stored. A replay returns it byte for byte with the header `Idempotent-Replayed: true`.
- **In flight.** A key that is still processing returns **409 `request_in_progress`**.
- **TTL.** API keys expire after 24 hours (configurable). Ledger-level keys are **permanent**: `binance:pay:*`, `binance:payout:*`, `sub:<id>:<period_start>`, `cs:<id>`.
- Every mutating endpoint requires the `Idempotency-Key` header.

### 2.7 API Conventions [PREMIUM]

- **Prefixed public IDs.** `usr_`, `acct_`, `tx_`, `wd_`, `dep_`, `prod_`, `cs_`, `sub_`, `whd_`, followed by a UUIDv7.
- **Amounts on the wire.** Minor-unit integers **as strings**, for example `"amount": "1000000", "currency": "FIN"`.

  - JavaScript parses them with `BigInt`.
  - The frontend never does money arithmetic in `Number`.
  - Formatting uses `Intl.NumberFormat` on BigInt.
- **Error envelope:**

  ```json
  {"error":{"type":"invalid_request","code":"amount_below_minimum","message":"…","param":"amountFin","request_id":"req_…"}}
  ```
- **Request IDs.** Every response carries a `Request-Id` header, and logs are keyed on it.
- **Versioning.** The header `Fynvar-Version: 2026-10-01` gives dated, non-breaking evolution, following Stripe's model.
- **Test mode.**

  - Keys are `fyn_test_…` or `fyn_live_…`, and every object carries a `livemode` flag.
  - Test mode runs against the Binance sandbox and a separate SQLite file.
  - Merchants can't integrate safely without it.
- **API keys.**

  - Format: `fyn_live_<8-char prefix>_<32 random>`.
  - Store `HMAC-SHA256(pepper, key)`, with the pepper from an environment variable. Store the prefix in plain text so keys can be identified in the UI.
  - Scopes: `checkout:write`, `products:write`, `refunds:write`, `read_only`.
- **Outgoing webhook signatures.**

  - Header: `Fynvar-Signature: t=<unix>,v1=<hex hmac>`, where `v1 = HMAC(secret, t + "." + rawBody)`.
  - Include the delivery ID.
  - Ship a 10-line PHP and Node verification snippet in `merchant.html`.
- **SSRF protection [HARDEN].** Merchant webhook URLs must be HTTPS. Resolve DNS at send time and refuse private, loopback and link-local ranges. Follow no redirects and cap the response at 4 KB.

### 2.8 Merchants, Checkout, Subscriptions and Refunds

**Checkout.**

- The price comes from the product and is **snapshotted into the checkout session**. It is never taken from the merchant's request.
- The buyer handle is optional; requiring it would leak handles.
- `success_url` and `cancel_url` must match the merchant's **registered domains**, which closes the open-redirect hole.
- `checkout.html` sends `Content-Security-Policy: frame-ancestors 'none'` to block clickjacking.
- Sessions move through `open → paid | expired | cancelled`.
- Confirmation uses the idempotency key `cs:<id>`, so a session can only ever be paid once.

**Subscriptions as mandates.**

- The buyer authorizes a mandate: merchant, product, maximum amount per period and interval.
- If the merchant raises the price, the buyer must consent again. The cron never charges more than the mandate allows.
- Each renewal uses the idempotency key `sub:<id>:<period_start>`, so a cron that runs twice can't charge twice.
- A pre-renewal reminder email is sent, and failures enter a dunning sequence.

**Refunds.**

- Refunds come from the merchant's FIN balance.
- If the balance is too low, the configured policy applies:
  - refuse the refund, or
  - allow a monitored negative balance backed by the merchant's rolling reserve (for example, 5% held for 30 days).
- The rolling reserve is the standard protection against merchants who disappear.

**Merchant onboarding.**

- KYB (business identity) runs when `admin-approval` is set.
- Merchants pick payouts in FIN (an internal transfer) or USDT (which reuses the withdrawal pipeline).

**Split payments [PREMIUM][METAX].**

- `application_fee_bps` per checkout lets a *platform* merchant take a cut while the seller receives the rest, all in one balanced journal.
- This is the Stripe Connect pattern, and it fits the Academies Community directly.

**Merchant webhook events:** `payment.succeeded`, `payment.failed`, `payment.refunded`, `subscription.created`, `subscription.renewed`, `subscription.payment_failed`, `subscription.cancelled`, `payout.completed`.

### 2.9 Settings Architecture [FIX]

- **The registry lives in code** (`settings_registry.php`). Each entry has:
  - key, group, type, default and validator,
  - cross-field rules,
  - flags: `sensitive`, `immutable_after_launch`, `requires_dual_approval`.
- **The database stores only overrides.** "Use the default, never a hardcoded fallback" then has one source of truth.
- **Cache.** One query per request into a static array, plus a `settings_version` counter in the database. Optionally, APCu with version-checked invalidation.
- **Secrets.**
  - The root encryption key lives **only** in the environment.
  - Binance and S3 secrets should also come from the environment.
  - If they are stored in the database, encrypt them with a key-ID prefix (`k1:` + nonce + ciphertext) so keys can be rotated.
  - The admin UI shows "configured ✓ · last rotated …" and never displays the value.
- **Effective-dated fees.**
  - A fee change takes effect at `effective_at`, which must be at least the notice period after the change (for example, 30 days for increases).
  - The public rates page shows both the current and the scheduled fee.
  - This is a consumer-protection requirement in many jurisdictions.
- **Cross-field rules (enforced on save):**
  - `max_withdrawal ≤ min(daily_withdrawal_cap, 80k USD equivalent)`. 80k USD = 800,000,000 FIN minor.
  - `auto_approve_threshold ≥ min_withdrawal`. Otherwise warn, because threshold mode would never auto-approve.
  - `max_deposit ≥ min_deposit`, and `max_transfer ≥ min_transfer`.
  - `alert_threshold ≥ min_reserve_ratio`.
  - Retry interval list length ≥ retry attempts.
  - 2FA methods must be non-empty when the policy is not `optional`.
  - S3 fields are required when the backup destination is `s3`.
  - Peg, FIN decimals and USDT decimals are rejected if any ledger entry exists.

### 2.10 Testing Strategy

**Core tests:**

- Unbalanced journal rejection.
- Idempotency replay, plus 422 on fingerprint mismatch and 409 while in flight.
- Concurrent transfers prevent double-spend.
- Deposit credit idempotency on webhook replay.
- Fee computation edge cases (rounding modes, minimums).
- Limit enforcement (daily and weekly caps, min/max).
- Withdrawal approval modes.
- Subscription renewal success and failure paths.
- Binance webhook signature verification (valid and invalid).
- Reconciliation detects a tampered balance.

**v1.1 additions:**

- **Property-based ledger tests.** Generate thousands of random journals and check that I1–I6 always hold.
- **Real concurrency.** Fork 50 PHP processes that each try to spend the same balance. Exactly the allowed number must succeed.
- **Webhook chaos:**
  - duplicates,
  - out-of-order delivery,
  - `PAY_SUCCESS` before the order row is committed,
  - an unknown cert SN,
  - a tampered body,
  - a stale timestamp,
  - a millisecond-versus-second timestamp mix-up.
- **Payout failure paths.** Each terminal and retryable error code ends in exactly one SETTLE or one RELEASE, never both.
- **Golden fee tables.** CSV fixtures for every fee type, rounding mode and minimum. Include the ₣1 transfer whose 0.25% fee rounds to zero.
- **Settings validation.** Every cross-field rule rejects bad combinations.
- **Hash chain.** Changing one historical entry makes reconciliation fail.
- **Append-only triggers.** UPDATE and DELETE on ledger tables abort.
- **Restore drill.** A restore from backup reproduces the published chain head.

---

## Part 3 — Admin Configuration Catalog

### How the catalog works

- Every field below is a **form field in the admin panel**.
- Each field has a type, a default and a validation rule, and every change is audited.
- Defaults live in the code registry. The database stores overrides.
- The UI groups fields into tabs that match the groups below.
- A versioned read cache means settings are not queried on every request.

**Flags:**

| Flag | Meaning                                                                          |
| ---- | -------------------------------------------------------------------------------- |
| 🔒   | Immutable after the first ledger entry (launch)                                  |
| 👥   | Requires dual approval (maker–checker)                                          |
| 🔑   | Sensitive (environment variable preferred; encrypted if stored; never displayed) |
| 🔐   | Locked value (shown but not editable)                                            |

**FIN minor-unit reference:** 1 FIN = 100 minor units, and 1 USDT = 10,000 FIN minor units. So 1,000,000 FIN minor = 10,000 FIN = $100.

### Group 1 — Currency and Peg

| Field                 | Type                  | Default | Validation                                   | Flags |
| --------------------- | --------------------- | ------- | -------------------------------------------- | ----- |
| FIN per USDT          | number                | 100     | > 0                                          | 🔒 👥 |
| FIN decimals          | integer               | 2       | 0–8                                         | 🔒 👥 |
| USDT decimals         | integer               | 6       | 0–8                                         | 🔒 👥 |
| Peg mode              | select: fixed / admin | fixed   | `dynamic` removed                          | 🔒 👥 |
| Currency display name | text                  | Finvar  | 1–32 chars                                  |       |
| Currency ticker       | text                  | FIN     | 2–6 uppercase                               |       |
| Currency symbol       | text                  | ₣      | 1–4 chars (decorative; UI leads with "FIN") |       |

### Group 2 — Fees

All fee changes are **effective-dated** (see Group 21 for notice days) and require dual approval. Fee payer is configured **per fee type**. Split rounding remainders go to the platform, and this is documented.

| Field                       | Type                               | Default  | Validation                                                   | Flags |
| --------------------------- | ---------------------------------- | -------- | ------------------------------------------------------------ | ----- |
| Deposit fee (bps)           | integer                            | 0        | 0–10000; price to cover processor fees if they are absorbed | 👥    |
| Deposit fee min (minor)     | integer                            | 0        | ≥ 0                                                         | 👥    |
| Transfer fee (bps)          | integer                            | 25       | 0–10000                                                     | 👥    |
| Transfer fee min (minor)    | integer                            | 0        | ≥ 0                                                         | 👥    |
| Transfer fee payer          | select: sender / recipient / split | sender   | —                                                           | 👥    |
| Withdrawal fee (bps)        | integer                            | 100      | 0–10000                                                     | 👥    |
| Withdrawal fee min (minor)  | integer                            | 5000     | ≥ 0                                                         | 👥    |
| Merchant checkout fee (bps) | integer                            | 150      | 0–10000 (single source; referenced by Group 7)              | 👥    |
| Checkout fee payer          | select: merchant / buyer / split   | merchant | —                                                           | 👥    |
| Subscription fee (bps)      | integer                            | 150      | 0–10000                                                     | 👥    |
| Refund fee (bps)            | integer                            | 0        | 0–10000 (single source; referenced by Group 10)             | 👥    |
| Payout fee (bps)            | integer                            | 0        | 0–10000                                                     | 👥    |
| Fee split (%)               | integer                            | 50       | 0–100; only where a payer is `split`                      | 👥    |
| Rounding mode               | select: half-up / floor / ceiling  | half-up  | —                                                           | 👥    |

> The "Checkout fee payer" options and default are illustrative of the per-fee-type rule.

### Group 3 — Limits and Thresholds

| Field                             | Type    | Default                  | Validation                                                              |
| --------------------------------- | ------- | ------------------------ | ----------------------------------------------------------------------- |
| Min deposit (USDT)                | number  | 100                      | > 0                                                                     |
| Max deposit (USDT)                | number  | 100000                   | ≥ min                                                                  |
| Deposit step (USDT)               | number  | 100                      | > 0                                                                     |
| Min withdrawal (FIN minor)        | integer | 1000000 ($100)           | > 0                                                                     |
| Max withdrawal (FIN minor)        | integer | **50000000** ($5k) | ≥ min; ≤ min(daily cap, 800000000 = 80k USD Binance per-detail limit) |
| Daily withdrawal cap (FIN minor)  | integer | 50000000                 | ≥ 0                                                                    |
| Weekly withdrawal cap (FIN minor) | integer | 200000000                | ≥ daily cap                                                            |
| Min transfer (FIN minor)          | integer | 100                      | ≥ 1                                                                    |
| Max transfer per tx (FIN minor)   | integer | 100000000                | ≥ min                                                                  |
| Daily transfer cap (FIN minor)    | integer | 1000000000               | ≥ 0                                                                    |
| Max balance per user (FIN minor)  | integer | 10000000000              | ≥ 0; enforced at deposit order creation                                |

### Group 4 — Withdrawal Controls

| Field                                   | Type                             | Default                  | Validation                                              | Flags |
| --------------------------------------- | -------------------------------- | ------------------------ | ------------------------------------------------------- | ----- |
| Approval mode                           | select: auto / admin / threshold | admin                    | —                                                      | 👥    |
| Auto-approve threshold (FIN minor)      | integer                          | **5000000** ($500) | ≥ min withdrawal (warn otherwise); threshold mode only | 👥    |
| Dual approval above (FIN minor)         | integer                          | 50000000                 | ≥ 0                                                    | 👥    |
| New payout destination cooldown (hours) | integer                          | 24                       | 0–720                                                  |       |
| Require payout destination allowlist    | boolean                          | true                     | —                                                      |       |
| Validate receiver via Binance           | boolean                          | true                     | —                                                      |       |
| Allowed receiver types                  | multi-select: BINANCE_ID / EMAIL | [BINANCE_ID, EMAIL]      | non-empty                                               |       |
| Require 2FA for withdrawals             | boolean                          | true                     | —                                                      |       |
| Require email confirmation              | boolean                          | false                    | —                                                      |       |
| Max pending withdrawals per user        | integer                          | 3                        | ≥ 1                                                    |       |
| Processing window start (UTC hour)      | integer                          | 0                        | 0–23                                                   |       |
| Processing window end (UTC hour)        | integer                          | 24                       | 0–24                                                   |       |
| Batch interval (minutes)                | integer                          | 60                       | ≥ 5                                                    |       |
| Payout status poll interval (minutes)   | integer                          | 5                        | ≥ 1                                                    |       |

> "Dual approval above" and "Payout status poll interval" are implementation additions. They support the maker–checker threshold and Payout Query polling required by §2.4–§2.5.

### Group 5 — Deposit Controls

The on/off switch for deposits moves to Group 17.

| Field                                          | Type                          | Default   | Validation                      | Flags |
| ---------------------------------------------- | ----------------------------- | --------- | ------------------------------- | ----- |
| Binance environment                            | select: sandbox / production  | sandbox   | —                              | 👥    |
| Binance API key                                | secret                        | —        | non-empty when deposits enabled | 🔑    |
| Binance secret key                             | secret                        | —        | non-empty when deposits enabled | 🔑    |
| Certificate auto-fetch (Query Certificate API) | boolean                       | true      | —                              |       |
| Fallback Binance public key                    | text (PEM)                    | —        | valid PEM if set                |       |
| Order expiry (minutes)                         | integer                       | 30        | **5–60**                 |       |
| Auto-credit on verified webhook                | boolean                       | true      | —                              |       |
| Re-query order before credit                   | boolean                       | true      | —                              | 🔐    |
| Exact amount + currency match                  | boolean                       | true      | —                              | 🔐    |
| Manual review threshold (USDT)                 | number                        | 10000     | ≥ 0                            |       |
| Processor fee handling                         | select: absorb / pass_to_user | absorb    | —                              | 👥    |
| Additional rails enabled                       | multi-select                  | [binance] | subset of known rails           |       |

> **Removed:** "Required confirmations" (Binance Pay is off-chain) and "Binance Pay enabled" (moved to Group 17).

### Group 6 — Transfers

The on/off switch for transfers moves to Group 17.

| Field                             | Type                          | Default                                | Validation                         |
| --------------------------------- | ----------------------------- | -------------------------------------- | ---------------------------------- |
| Recipient identifier              | select: handle / email / both | handle                                 | —                                 |
| Handle change cooldown (days)     | integer                       | 30                                     | 0–365                             |
| Released handle quarantine (days) | integer                       | 90                                     | 0–365                             |
| Reserved handles                  | CSV                           | admin, support, fynvar, metax, binance | each matches `^[a-z0-9_]{3,20}$` |
| Require recipient to exist        | boolean                       | true                                   | —                                 |
| Allow self-transfer               | boolean                       | false                                  | —                                 |
| Memo required                     | boolean                       | false                                  | —                                 |
| Memo max length                   | integer                       | 140                                    | 0–500                             |
| Lookup requests / min / user      | integer                       | 20                                     | ≥ 1                               |

### Group 7 — Merchant Platform

The merchant fee is defined in Group 2. Webhook retry settings are defined in Group 16.

| Field                             | Type                                         | Default        | Validation                       |
| --------------------------------- | -------------------------------------------- | -------------- | -------------------------------- |
| Merchant onboarding               | select: auto / admin-approval (KYB)          | admin-approval | —                               |
| Payout currency                   | select: FIN / USDT / both                    | both           | —                               |
| Min payout (FIN minor)            | integer                                      | 1000000        | ≥ 0                             |
| Payout schedule                   | select: on-demand / daily / weekly / monthly | weekly         | —                               |
| Payout day                        | integer                                      | 1              | 1–7 (weekly) or 1–28 (monthly) |
| Rolling reserve (%)               | integer                                      | 5              | 0–100                           |
| Rolling reserve hold (days)       | integer                                      | 30             | 0–180                           |
| Registered redirect domains       | CSV per merchant                             | —             | valid hostnames                  |
| Split payments allowed            | boolean                                      | true           | —                               |
| Max application fee (bps)         | integer                                      | 3000           | 0–10000                         |
| API key rotation (days)           | integer                                      | 90             | 0 (never) or ≥ 7                |
| Max API keys per merchant         | integer                                      | 5              | 1–20                            |
| Webhook signing algorithm         | select: hmac-sha256 / hmac-sha512            | hmac-sha256    | —                               |
| Webhook timeout (seconds)         | integer                                      | 10             | 1–60                            |
| Webhook max response size (KB)    | integer                                      | 4              | 1–64                            |
| Checkout session expiry (minutes) | integer                                      | 30             | 5–1440                          |
| Checkout branding                 | select: fynvar / merchant                    | fynvar         | —                               |
| Allow merchant custom domain      | boolean                                      | false          | —                               |

### Group 8 — Products and Checkout

| Field                                     | Type                                    | Default       | Validation                     |
| ----------------------------------------- | --------------------------------------- | ------------- | ------------------------------ |
| Max products per merchant                 | integer                                 | 100           | 1–10000                       |
| Product price min (FIN minor)             | integer                                 | 100           | ≥ 1                           |
| Product price max (FIN minor)             | integer                                 | 1000000000    | ≥ min                         |
| Allow recurring products                  | boolean                                 | true          | —                             |
| Allowed intervals                         | multi-select: day / week / month / year | [month, year] | non-empty if recurring allowed |
| Trial period allowed                      | boolean                                 | true          | —                             |
| Trial max days                            | integer                                 | 30            | 0–365                         |
| Checkout require login                    | boolean                                 | true          | —                             |
| Guest checkout allowed                    | boolean                                 | false         | —                             |
| Checkout redirect URL required            | boolean                                 | true          | must match registered domains  |
| Buyer handle required on session creation | boolean                                 | false         | —                             |

### Group 9 — Subscriptions

| Field                                      | Type                                | Default   | Validation                   |
| ------------------------------------------ | ----------------------------------- | --------- | ---------------------------- |
| Renewal grace period (hours)               | integer                             | 24        | 0–720                       |
| Max retry attempts                         | integer                             | 3         | 0–10                        |
| Retry intervals (hours, CSV)               | CSV of ints                         | 24,72,168 | each ≥ 1; count ≥ attempts |
| On final failure                           | select: cancel / suspend / past_due | cancel    | —                           |
| Price increase requires mandate re-consent | boolean                             | true      | 🔐                           |
| Proration on upgrade                       | boolean                             | true      | —                           |
| Proration on downgrade                     | boolean                             | false     | —                           |
| Cancel at period end                       | boolean                             | true      | —                           |
| Allow immediate cancel                     | boolean                             | true      | —                           |
| Refund on cancel                           | boolean                             | false     | —                           |
| Renewal reminder (days before)             | integer                             | 3         | 0–30                        |

### Group 10 — Refunds

The refund fee is defined in Group 2.

| Field                                  | Type                                     | Default | Validation |
| -------------------------------------- | ---------------------------------------- | ------- | ---------- |
| Refunds enabled                        | boolean                                  | true    | —         |
| Refund window (days)                   | integer                                  | 30      | 0–365     |
| Who initiates                          | select: merchant / admin / both          | both    | —         |
| Partial refunds allowed                | boolean                                  | true    | —         |
| Refund to original balance only        | boolean                                  | true    | —         |
| Auto-approve refunds under (FIN minor) | integer                                  | 100000  | ≥ 0       |
| Insufficient merchant balance policy   | select: refuse / reserve_backed_negative | refuse  | —         |

### Group 11 — Reserve and Treasury

| Field                                         | Type                   | Default  | Validation | Flags |
| --------------------------------------------- | ---------------------- | -------- | ---------- | ----- |
| Min reserve ratio (%)                         | integer                | 100      | 0–200     | 👥    |
| Alert threshold (%)                           | integer                | 105      | ≥ min     |       |
| Reconciliation schedule                       | select: hourly / daily | daily    | —         |       |
| Auto-pause deposits on breach                 | boolean                | true     | —         |       |
| Auto-pause withdrawals on breach              | boolean                | true     | —         |       |
| Treasury external address / account reference | text                   | —       | optional   |       |
| Large deposit alert (USDT)                    | number                 | 10000    | ≥ 0       |       |
| Large withdrawal alert (FIN minor)            | integer                | 10000000 | ≥ 0       |       |

### Group 12 — KYC and AML

| Field                          | Type                             | Default | Validation  |
| ------------------------------ | -------------------------------- | ------- | ----------- |
| KYC required                   | boolean                          | false   | —          |
| KYC trigger threshold (USDT)   | number                           | 1000    | ≥ 0        |
| KYC provider                   | select: manual / sumsub / onfido | manual  | —          |
| Merchant KYB required          | boolean                          | true    | —          |
| Sanctions screening enabled    | boolean                          | false   | —          |
| Transaction monitoring enabled | boolean                          | false   | —          |
| Travel Rule data collection    | boolean                          | false   | —          |
| SAR email                      | email                            | —      | valid email |
| Blocked countries              | CSV of ISO codes                 | []      | valid codes |
| Blocked IPs                    | CSV of CIDRs                     | []      | valid CIDRs |

> "Merchant KYB required" and "Travel Rule data collection" are additions that support §1.10 and §2.8.

### Group 13 — Auth and Sessions

| Field                          | Type                                                          | Default                 | Validation                                                             | Flags |
| ------------------------------ | ------------------------------------------------------------- | ----------------------- | ---------------------------------------------------------------------- | ----- |
| Access token TTL (minutes)     | integer                                                       | 15                      | 1–1440                                                                |       |
| Refresh token TTL (days)       | integer                                                       | 30                      | 1–365                                                                 |       |
| Rotate refresh token on use    | boolean                                                       | true                    | —                                                                     | 🔐    |
| Revoke family on token reuse   | boolean                                                       | true                    | —                                                                     | 🔐    |
| Session idle timeout (minutes) | integer                                                       | 60                      | 5–1440                                                                |       |
| Admin session TTL (minutes)    | integer                                                       | 15                      | 5–60                                                                  |       |
| Admin IP allowlist             | CSV of CIDRs                                                  | []                      | valid CIDRs                                                            | 👥    |
| Password min length            | integer                                                       | 12                      | 8–128                                                                 |       |
| Password require upper         | boolean                                                       | true                    | —                                                                     |       |
| Password require lower         | boolean                                                       | true                    | —                                                                     |       |
| Password require digit         | boolean                                                       | true                    | —                                                                     |       |
| Password require symbol        | boolean                                                       | true                    | —                                                                     |       |
| Argon2 memory (MB)             | integer                                                       | 64                      | 16–1024; × max concurrent hashes < FPM budget                        |       |
| Argon2 time cost               | integer                                                       | 3                       | 1–10                                                                  |       |
| Argon2 parallelism             | integer                                                       | 4                       | 1–16;**effective value detected at runtime is shown alongside** |       |
| Max login attempts             | integer                                                       | 5                       | 1–20                                                                  |       |
| Lockout duration (minutes)     | integer                                                       | 15                      | 1–1440                                                                |       |
| Require email verification     | boolean                                                       | true                    | —                                                                     |       |
| 2FA policy                     | select: optional / required-for-withdrawal / required-for-all | required-for-withdrawal | —                                                                     |       |
| 2FA methods                    | multi-select: totp / email / passkey                          | [totp, passkey]         | non-empty when policy not optional;**sms removed**               |       |
| Admin requires passkey         | boolean                                                       | true                    | —                                                                     | 🔐    |
| Backup codes count             | integer                                                       | 10                      | 5–20                                                                  |       |
| Step-up 2FA for money actions  | boolean                                                       | true                    | —                                                                     |       |

### Group 14 — Rate Limiting

| Field                             | Type    | Default | Validation                            |
| --------------------------------- | ------- | ------- | ------------------------------------- |
| Auth requests / min / IP          | integer | 10      | ≥ 1; bounded by Argon2 memory budget |
| API requests / min / user         | integer | 120     | ≥ 1                                  |
| Transfer requests / min / user    | integer | 10      | ≥ 1                                  |
| Deposit orders / hour / user      | integer | 20      | ≥ 1                                  |
| Webhook requests / min / merchant | integer | 600     | ≥ 1                                  |
| Password reset / hour / email     | integer | 3       | ≥ 1                                  |

### Group 15 — Notifications

| Field                                                             | Type     | Default | Validation             | Flags |
| ----------------------------------------------------------------- | -------- | ------- | ---------------------- | ----- |
| Email enabled                                                     | boolean  | true    | —                     |       |
| SMTP host                                                         | text     | —      | non-empty when enabled |       |
| SMTP port                                                         | integer  | 587     | 1–65535               |       |
| SMTP user                                                         | text     | —      | —                     |       |
| SMTP password                                                     | secret   | —      | —                     | 🔑    |
| SMTP from address                                                 | email    | —      | valid email            |       |
| SMTP from name                                                    | text     | Fynvar  | —                     |       |
| Admin alert webhook URL                                           | url      | —      | valid HTTPS URL        |       |
| Welcome email template                                            | textarea | —      | —                     |       |
| Deposit confirmation template                                     | textarea | —      | —                     |       |
| Withdrawal status template (held / settled / released / rejected) | textarea | —      | —                     |       |
| Transfer received template                                        | textarea | —      | —                     |       |
| Subscription reminder template                                    | textarea | —      | —                     |       |
| Subscription renewal template                                     | textarea | —      | —                     |       |
| Subscription failure template                                     | textarea | —      | —                     |       |
| Security alert template                                           | textarea | —      | —                     |       |
| Fee change notice template                                        | textarea | —      | —                     |       |

### Group 16 — Idempotency, Replay and Webhook Delivery

This is the single source for webhook retry settings.

| Field                                           | Type    | Default        | Validation                                                |
| ----------------------------------------------- | ------- | -------------- | --------------------------------------------------------- |
| Idempotency key TTL (hours)                     | integer | 24             | 1–720 (ledger-level keys are permanent)                  |
| Incoming webhook replay window (seconds)        | integer | 300            | 30–3600; compared against**ms** timestamps × 1000 |
| Outgoing webhook retry attempts                 | integer | 5              | 0–20                                                     |
| Outgoing webhook retry intervals (minutes, CSV) | CSV     | 1,5,30,120,720 | each ≥ 1; count ≥ attempts                              |
| SQLITE_BUSY retry attempts                      | integer | 3              | 0–10                                                     |

### Group 17 — System and Feature Flags

There is one feature flag per module, all in this group.

| Field                               | Type     | Default         | Validation                | Flags |
| ----------------------------------- | -------- | --------------- | ------------------------- | ----- |
| Maintenance mode                    | boolean  | false           | —                        |       |
| Maintenance message                 | textarea | —              | —                        |       |
| Registration open                   | boolean  | true            | —                        |       |
| Invite-only mode                    | boolean  | false           | —                        |       |
| Feature: transfers                  | boolean  | true            | —                        |       |
| Feature: deposits (Binance Pay)     | boolean  | true            | requires Phase 0 complete |       |
| Feature: withdrawals                | boolean  | true            | —                        |       |
| Feature: merchant platform          | boolean  | false           | —                        |       |
| Feature: subscriptions              | boolean  | false           | —                        |       |
| Feature: refunds                    | boolean  | true            | —                        |       |
| Feature: referral program           | boolean  | false           | —                        |       |
| Referral reward (FIN minor)         | integer  | 5000            | ≥ 0                      |       |
| Referee reward (FIN minor)          | integer  | 5000            | ≥ 0                      |       |
| Referral commission (bps)           | integer  | 50              | 0–10000                  |       |
| Referral funding source             | select   | fee_revenue_fin | —                        | 🔐    |
| Monthly referral budget (FIN minor) | integer  | 1000000         | ≥ 0                      |       |

> The "Monthly referral budget" default is a placeholder. The operator must set it.

### Group 18 — Branding

| Field               | Type                          | Default                                  | Validation                                     |
| ------------------- | ----------------------------- | ---------------------------------------- | ---------------------------------------------- |
| Platform name       | text                          | Fynvar                                   | 1–64                                          |
| Platform tagline    | text                          | A balance built for software, not banks. | ≤ 160                                         |
| Logo URL            | url                           | —                                       | valid HTTPS URL (self-hosted origin under CSP) |
| Favicon URL         | url                           | —                                       | valid HTTPS URL                                |
| Primary color (hex) | text                          | #06caae                                  | valid hex; contrast-checked for button text    |
| Accent color (hex)  | text                          | #7c5cff                                  | valid hex                                      |
| Default theme       | select: dark / light / system | dark                                     | —                                             |
| Support email       | email                         | —                                       | valid email                                    |
| Terms URL           | url                           | —                                       | valid URL                                      |
| Privacy URL         | url                           | —                                       | valid URL                                      |

### Group 19 — Audit and Logging

| Field                        | Type                                | Default | Validation |
| ---------------------------- | ----------------------------------- | ------- | ---------- |
| Audit log retention (days)   | integer                             | 365     | 30–3650   |
| Log level                    | select: debug / info / warn / error | info    | —         |
| Log admin actions            | boolean                             | true    | —         |
| Log fee changes              | boolean                             | true    | —         |
| Log config changes           | boolean                             | true    | —         |
| Log failed logins            | boolean                             | true    | —         |
| Log maker–checker decisions | boolean                             | true    | 🔐         |

### Group 20 — Database and Backup

| Field                             | Type                             | Default   | Validation     | Flags |
| --------------------------------- | -------------------------------- | --------- | -------------- | ----- |
| Backup frequency                  | select: hourly / daily / weekly  | daily     | —             |       |
| Backup retention count            | integer                          | 14        | 1–365         |       |
| Backup destination                | select: local / s3 / gcs         | local     | —             |       |
| S3 bucket                         | text                             | —        | required if s3 |       |
| S3 region                         | text                             | —        | required if s3 |       |
| S3 access key                     | secret                           | —        | required if s3 | 🔑    |
| S3 secret key                     | secret                           | —        | required if s3 | 🔑    |
| WAL checkpoint interval (minutes) | integer                          | 15        | 1–1440        |       |
| Vacuum schedule                   | select: never / weekly / monthly | weekly    | —             |       |
| Restore drill schedule            | select: monthly / quarterly      | quarterly | —             |       |

### Group 21 — Disclosures (new) [METAX]

| Field                                       | Type                   | Default        | Validation  |
| ------------------------------------------- | ---------------------- | -------------- | ----------- |
| Fee-change notice (days, increases)         | integer                | 30             | ≥ 0        |
| Public reserve page enabled (`/reserves`) | boolean                | true           | —          |
| Reserve snapshot frequency                  | select: hourly / daily | hourly         | —          |
| Chain-head publishing                       | boolean                | true           | —          |
| Correction log URL                          | url                    | /corrections   | valid URL   |
| Declared-limits text                        | textarea               | (default copy) | —          |
| Unsupported jurisdictions                   | CSV of ISO codes       | []             | valid codes |

---

## Part 4 — Super AI Agent Prompt

Copy everything below into the AI agent. It is self-contained and already includes all v1.1 amendments.

---

**Act as a senior full-stack engineer and product architect.**

You are building **Fynvar**, a closed-loop digital payment platform with its own internal currency, the **Finvar (ticker: FIN)**, funded by USDT deposits through Binance Pay.

You will deliver a complete, production-shaped implementation:

- static HTML pages on `pay.metax.academy`, and
- a PHP + SQLite API whose backend lives at `gloulou.net/fynvar-api/`. It is exposed **same-site** as `pay.metax.academy/api/` through a reverse proxy, or as `api.pay.metax.academy`.

There is no blockchain. FIN exists only as entries in a double-entry ledger.

Build everything described below. Do not skip modules, and do not stub endpoints. Every feature must work end to end. If a third-party credential is missing (Binance keys, SMTP), implement the code path fully, read the credential from environment variables, fall back to sandbox, and show a clear error when it is unset.

> **Precedence.** This prompt is the consolidated v1.1 spec. Where anything here conflicts with older material, this prompt wins.

### A. Foundational rules

1. **All money is integers in minor units.**
   - FIN has 2 decimals; USDT has 6. 1 FIN minor = 100 USDT minor, exactly.
   - Never use floats for arithmetic.
   - On the wire, amounts are integer **strings**. The client parses them with `BigInt` and never does money arithmetic in `Number`.
2. **The ledger is double-entry.**
   - `amount > 0` increases a balance, and entries sum to zero per currency.
   - A single `postJournal` function does everything inside one database transaction: validates balance, enforces idempotency, inserts the transaction and entries, updates the balance cache and extends the hash chain.
   - Nothing else may write to `ledger_entries` or `account_balances`.
3. **Every table has an integer primary key plus a prefixed UUIDv7 public ID** (`usr_`, `acct_`, `tx_`, `wd_`, `dep_`, `prod_`, `cs_`, `sub_`, `whd_`).
   - External identifiers are always public IDs. Internal joins use integers.
   - Tables are `STRICT`.
   - Timestamps use `strftime('%Y-%m-%dT%H:%M:%fZ','now')`.
4. **Every mutating endpoint requires an `Idempotency-Key` header.**
   - Keys are scoped to `(principal, method, path, key)` and fingerprinted.
   - A replay returns the stored response with `Idempotent-Replayed: true`.
   - A fingerprint mismatch returns 422 `idempotency_key_reused`; a key still in flight returns 409 `request_in_progress`.
5. **Client-supplied amounts are never trusted for credits.**
   - Deposits credit only after a verified webhook **and** a successful Query Order re-check with an exact currency and amount match.
   - Checkout prices are snapshotted from the product.
   - For transfers, the client sends only `toHandle` and `amountFin`.
6. **Concurrency.** SQLite has no row locks.
   - Every money transaction uses `BEGIN IMMEDIATE` and guarded debits: `UPDATE … SET balance = balance - :amt WHERE … AND balance >= :amt`. Zero rows means insufficient funds.
   - Retry `SQLITE_BUSY` up to 3 times with jitter.
   - Write against PDO generically so a later PostgreSQL move is a driver swap.
7. **SQLite settings:** WAL mode, `busy_timeout = 5000`, `synchronous = NORMAL`, `foreign_keys = ON`.
8. **Sensitive data:**
   - Encrypt sensitive columns with libsodium secretbox, using a key-ID prefix (`k1:` + nonce + ciphertext) for rotation. This covers TOTP secrets, merchant webhook secrets and any stored provider secrets.
   - The root key lives only in the environment.
9. **Credentials:**
   - Passwords use Argon2id. Detect the backend (libsodium vs libargon2) at boot and log the effective parameters.
   - Merchant API keys are stored as `HMAC-SHA256(pepper, key)` and shown exactly once.
10. **Tamper evidence:**
    - Append-only triggers on `transactions` and `ledger_entries`.
    - A SHA-256 hash chain over transactions.
    - The daily chain head is published.
11. **Invariants I1–I6** (Part 2 §2.1.4) are asserted in tests and at reconciliation.
12. **No secrets in the repo.**
13. **Write all code yourself**, guided by this specification.

### B. Tech stack and file layout

- **Static pages:** HTML5, vanilla JS (ES2022 modules) and **prebuilt** Tailwind CSS (no CDN). No framework.
- **API:** PHP 8.2+, PDO + SQLite, no framework.
  - `/fynvar-api/` is canonical; `/fynval-api/` is a 308 alias.
- **Email:** SMTP via PHPMailer or a hand-rolled client.
- **Binance Pay:** Merchant API v2. HMAC-SHA512 for outgoing requests; RSA-SHA256 verification for incoming webhooks, with certificates from the Query Certificate API.

Create the exact directory layout in Part 1 §1.13. Every file listed there must exist and work.

### C. Data model

Implement migrations for these tables:

- **Core:** `users`, `handle_history`, `accounts`, `transactions` (with `prev_hash`, `hash`), `ledger_entries`, `account_balances`.
- **Settings and approvals:** `settings_overrides`, `settings_meta` (`settings_version`), `scheduled_fee_changes`, `approvals` (maker–checker).
- **Merchants and commerce:** `merchants`, `merchant_domains`, `api_keys`, `products`, `checkout_sessions`, `mandates`/`subscriptions`, `refunds`.
- **Money movement:** `withdrawals`, `payout_destinations`, `payouts`, `deposit_orders`.
- **Binance:** `binance_certificates`, `webhook_nonces`.
- **Delivery and API:** `webhook_deliveries`, `idempotency_keys`.
- **Auth:** `refresh_tokens` (with `family_id`), `totp_secrets`, `backup_codes`, `webauthn_credentials`, `sessions`, `login_attempts`, `rate_limit_buckets`.
- **Audit and transparency:** `audit_log`, `reconciliation_runs`, `chain_anchors`, `correction_log`.

Every table is `STRICT` and has an integer primary key and a public ID where it is exposed externally. Money columns get `CHECK (typeof(amount)='integer')`. JSON metadata goes in TEXT columns, queried with `json_extract()`.

Seed these system accounts: `system:treasury`, `system:mint_usdt`, `system:mint_fin`, `system:withdrawal_hold`, `system:payout_clearing`, `system:fee_revenue`, `system:fee_revenue_fin`, `system:processor_fees`, `system:merchant_reserve`, `system:adjustments`.

**Settings are not seeded into the database.** Defaults live in `settings_registry.php`, which mirrors the Part 3 catalog exactly: 21 groups, with types, defaults, validators, cross-field rules and flags. The database stores overrides only.

### D. Money handling

Write `money.php` with:

- human-to-minor conversion per currency,
- minor-to-human formatting,
- basis-point fee application (`applyBps`) with half-up, floor or ceiling rounding and fee minimums,
- exact FIN↔USDT minor conversion (× 100),
- overflow-safe 64-bit integer helpers that reject overflow.

Split-fee rounding remainders go to the platform.

### E. Ledger

Write `ledger.php` with a single `postJournal` function. It must:

- Accept a transaction kind, entries (account ID, currency, signed minor amount), an optional idempotency key, optional metadata and an optional existing PDO handle, so it can be composed inside a caller's `BEGIN IMMEDIATE`.
- Validate that entries balance to zero per currency, throwing `UnbalancedJournal` otherwise.
- Return the existing transaction for a known ledger-level key.
- Perform guarded debits on customer accounts, throwing `InsufficientFunds`.
- Insert the transaction and entries, upsert balances, and compute `hash = SHA-256(prev_hash ‖ canonical_json(tx + entries))`.

Implement the journal shapes in Part 2 §2.1.3 **exactly**:

- DEPOSIT (with processor fee and optional deposit fee)
- TRANSFER
- CHECKOUT/SUBSCRIPTION CHARGE (with platform fee, application fee and rolling reserve)
- REFUND
- WITHDRAWAL.HOLD / SETTLE / RELEASE
- REFERRAL REWARD (funded from `fee_revenue_fin`)
- manual ADJUSTMENT (against `system:adjustments` only)

Comment every financial operation with a one-line description of its entry structure.

Also write `accounts.php` with:

- get a system account by code,
- get or create a user's or merchant's account per currency,
- resolve an account by handle,
- a guarded debit helper.

### F. Authentication

Implement signup, login, refresh, logout, password reset, email verification, TOTP enrollment and verification, backup codes, and WebAuthn passkeys.

**Handles:**

- Validate against `^[a-z0-9_]{3,20}$`, case-folded.
- Enforce the reserved list and the 90-day quarantine for released handles.

**Sessions:**

- The access token is an Ed25519-signed JWT (or an opaque token) returned in the JSON body and held in JS memory, with a 15-minute TTL.
- The refresh token goes in `__Host-fyn_rt; Secure; HttpOnly; SameSite=Strict; Path=/`.
  - It is rotated on every use, in token families.
  - Reuse revokes the family and sends a security alert.
- `/auth/refresh` and `/auth/logout` require an allowlisted `Origin` and `X-Fynvar-Client: web`.

**TOTP:**

- ±1 step, with the last used step stored to block reuse.

**Login protection:**

- Rate-limit by IP and email.
- Lock out after the configured number of failures.

**Step-up and admins:**

- Require step-up 2FA for withdrawals, payout destination changes and API key creation.
- Admins must use passkeys and have 15-minute sessions, with an optional IP allowlist.

### G. Wallet endpoints

- `GET /wallet` returns the FIN balance, USDT balance (as integer strings) and handle.
- `GET /wallet/transactions` returns cursor-paginated history.
- `GET /wallet/transactions/:id` returns the transaction with its entries (filtered to the user's accounts), its state timeline and its chain hash.
- `GET /wallet/transactions/:id/receipt` returns receipt data.

### H. Transfers

`POST /transfers` accepts `toHandle` and `amountFin`. Inside `BEGIN IMMEDIATE`, it:

1. resolves the recipient,
2. rejects self-transfers,
3. checks limits,
4. computes the fee per settings (including fee payer),
5. performs a guarded debit of the sender,
6. posts the TRANSFER journal.

It returns the transaction ID, gross, fee and net.

`GET /transfers/lookup?handle=` is rate-limited and returns only display name and avatar.

### I. Deposits via Binance Pay

**Creating an order.** `POST /deposits` accepts an amount in USDT. It:

1. validates min, max and step,
2. enforces `max_balance` up front,
3. creates a Binance Pay order with a 32-character alphanumeric `merchantTradeNo` and `orderExpireTime` within 5–60 minutes,
4. persists a pending `deposit_orders` row,
5. returns the checkout URL, QR link and deep link.

**Signing outgoing requests:**

- Headers: `BinancePay-Timestamp` (ms), `BinancePay-Nonce` (32 letters), `BinancePay-Certificate-SN`, `BinancePay-Signature`.
- Signature: `strtoupper(hash_hmac('sha512', ts."\n".nonce."\n".body."\n", secret))` over the exact bytes sent.

**`webhooks/binance.php` must:**

- Read the raw body and headers.
- Fetch and cache the public key by Certificate-SN through Query Certificate, refreshing on an unknown SN and falling back to the configured PEM.
- Verify RSA-SHA256 over `ts\nnonce\nbody\n` against the base64-decoded signature. Return 401 on failure.
- Reject stale timestamps using millisecond normalization: `abs(now_ms − ts_ms) <= window_s × 1000`.
- Reject replayed `(cert_sn, nonce)` pairs.
- Decode `data` twice. Branch on `bizType`/`bizStatus` (`PAY_SUCCESS`, `PAY_CLOSED`, and payout events).
- On `PAY_SUCCESS`:
  1. Look up the order. If it isn't found, schedule a Query Order re-check.
  2. **Re-query the order.** Require success status, `currency == "USDT"` and an exact amount match.
  3. Hold for manual review if the amount is above the review threshold or anything mismatches.
  4. Post the DEPOSIT journal with the processor fee, keyed `binance:pay:<merchantTradeNo>`.
  5. If a credit would exceed `max_balance`, credit it, freeze the excess and open a review.
  6. Mark the order as posted.
- Always reply `{"returnCode":"SUCCESS","returnMessage":null}` on handled paths.

`GET /deposits/:merchantTradeNo` returns the deposit's status.

### J. Withdrawals

**Destinations.**

- `POST /destinations` adds a payout destination (`receiveType` ∈ BINANCE_ID/EMAIL, plus `receiver`).
- It validates the receiver via Binance's Payout Validate Receiver API, requires step-up 2FA, and starts the cooldown.

**Requesting a withdrawal.** `POST /withdrawals` accepts an amount in FIN and a destination ID. It:

1. validates min, max, daily and weekly limits, plus max pending,
2. checks the destination is allowlisted and past its cooldown,
3. requires step-up 2FA,
4. posts **WITHDRAWAL.HOLD** immediately.

**Approval** follows the configured mode: auto, admin, or threshold (with maker–checker above the dual-approval threshold).

**Payout.**

- Approved withdrawals are sent through **Batch Payout**: 32-character alphanumeric `requestId` and `merchantSendId`, `bizScene=MERCHANT_PAYMENT`, `transferMethod=FUNDING_WALLET`, and at most 80K USD per detail.
- `cron/poll-payouts.php` and the payout webhook both drive the outcome:
  - **SETTLE** on success.
  - **RELEASE** on rejection or terminal codes `400703`, `400709`, `400710`, `400711`, `406103`, `406106`.
  - **Retry** on `400000` and `400501`.
- Exactly one terminal outcome per withdrawal.

**Read endpoints:**

- `GET /withdrawals/:id` returns status.
- `GET /admin/withdrawals?status=pending` lists the queue, with bulk actions.

### K. Merchant platform

- `POST /merchants` registers a merchant.
  - KYB runs when onboarding mode is `admin-approval`.
  - Generates an encrypted webhook signing secret.
- `POST /api-keys` creates `fyn_test_…` or `fyn_live_<prefix>_<random>` keys with scopes.
  - Each key is shown once and stored as a peppered HMAC.
  - Requires step-up 2FA.
- `GET /merchants/me` returns the profile without secrets.
- `PATCH /merchants/me` updates name, webhook URL (HTTPS), registered redirect domains and payout preferences.
- `POST/GET/PATCH/DELETE /products` manage products. Validate min/max price, allowed intervals and trial settings.

Test mode uses the Binance sandbox and `fynvar_test.sqlite`. Every object carries `livemode`.

### L. Checkout

`POST /checkout` accepts:

- a product ID,
- an optional buyer handle,
- `success_url` and `cancel_url` (which must match registered domains),
- an optional `application_fee_bps` (≤ max) for split payments.

It snapshots the price, creates a session with the configured expiry, and returns `pay.metax.academy/checkout.html?c=<cs_id>`.

- `GET /checkout/:id` returns the render data.
- `POST /checkout/:id/confirm`, authenticated as the buyer and keyed `cs:<id>`:
  1. posts the CHECKOUT journal (platform fee, application fee, rolling reserve),
  2. moves the session `open → paid`,
  3. dispatches `payment.succeeded`.

`checkout.html` sends `frame-ancestors 'none'`.

### M. Subscriptions, refunds and cron

**Subscriptions (mandates):**

- `POST /subscriptions` creates a **mandate**: merchant, product, max amount per period, interval.
- `GET /subscriptions/me` lists the user's subscriptions.
- `POST /subscriptions/:id/cancel` cancels at period end or immediately, per settings.
- A price increase requires re-consent.

**Refunds:**

- `POST /refunds` (merchant or admin, per settings) handles full or partial refunds within the window.
- It applies the insufficient-balance policy and posts the reverse journal.
- It dispatches `payment.refunded`.

**Cron jobs:**

| Script                               | Purpose                                                                                                                                                                                                                                                                                                             |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cron/charge-subscriptions.php`    | Charges due mandates with key `sub:<id>:<period_start>`, never above the mandate maximum. Success advances the period and fires `subscription.renewed`. Failure schedules a retry and fires `subscription.payment_failed`. After max retries it applies the final state and fires `subscription.cancelled`. |
| `cron/send-renewal-reminders.php`  | Sends pre-renewal reminder emails.                                                                                                                                                                                                                                                                                  |
| `cron/retry-webhooks.php`          | Retries failed merchant webhooks per Group 16.                                                                                                                                                                                                                                                                      |
| `cron/expire-pending.php`          | Expires checkout sessions and deposit orders past their expiry.                                                                                                                                                                                                                                                     |
| `cron/reconcile.php`               | Checks I1–I6, re-walks the hash chain and compares treasury against Binance custody (± clearing). Logs mismatches, writes to the correction log, fires the admin webhook and auto-pauses on reserve breach.                                                                                                       |
| `cron/anchor-chain-head.php`       | Publishes the daily chain-head hash.                                                                                                                                                                                                                                                                                |
| `cron/apply-scheduled-fees.php`    | Activates effective-dated fee changes.                                                                                                                                                                                                                                                                              |
| `cron/release-rolling-reserve.php` | Releases matured rolling-reserve holds.                                                                                                                                                                                                                                                                             |
| `cron/backup-db.php`               | Takes timestamped backups, applies retention, optionally uploads to S3/GCS, and supports the restore drill.                                                                                                                                                                                                         |

### N. Merchant webhooks (outgoing)

The dispatcher:

- signs payloads as `Fynvar-Signature: t=<unix>,v1=<hex HMAC(secret, t + "." + rawBody)>`, using SHA-256 or SHA-512 per settings,
- includes a delivery ID,
- applies the SSRF guard: HTTPS only, DNS resolved at send time, private/loopback/link-local ranges refused, no redirects, 4 KB response cap,
- enforces the timeout,
- logs every attempt in `webhook_deliveries`.

Events: `payment.succeeded`, `payment.failed`, `payment.refunded`, `subscription.created`, `subscription.renewed`, `subscription.payment_failed`, `subscription.cancelled`, `payout.completed`.

Ship PHP and Node verification snippets in `merchant.html`.

### O. Admin panel

`admin.html` has a separate login gated by `is_admin` plus a passkey, a 15-minute session and an optional IP allowlist.

**Settings tabs:**

- Tabs match the **21 settings groups** in Part 3.
- Each field is rendered from the registry, with its type, default, validator, cross-field rules and flags.
- Sensitive values show only "configured ✓ · last rotated …".
- Immutable-after-launch fields lock once the first ledger entry exists.
- Flagged changes create a maker–checker proposal that a different admin must approve.
- Saving writes an override, bumps `settings_version` and writes an audit entry with actor(s), old value, new value, reason and timestamp.
- Fee changes are effective-dated.

**Other pages:**

- user list (search, impersonation off by default),
- transaction browser with filters,
- withdrawal queue with bulk actions,
- merchant list with KYB controls,
- fee history (current and scheduled),
- reserve dashboard (I3–I5),
- reconciliation runs,
- correction log editor,
- manual adjustment tool (against `system:adjustments`, reason required, maker–checker).

### P. Settings cache

Load all overrides once per request into a static array merged over registry defaults. Provide typed getters. Invalidate through `settings_version`; APCu is optional. Never query the table per key.

### Q. Static frontend pages

| Page                              | Contents                                                                                                                                           |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `index.html`                    | Gradient hero, the 100 USDT → 10,000 FIN exchange card, feature cards,**declared-limits section**, footer.                                  |
| `login.html`                    | Combined sign in / sign up with tabs, passkey support.                                                                                             |
| `dashboard.html`                | Balance card with count-up, deposit/send/withdraw actions, paginated activity, toasts, trust micro-copy, state badges.                             |
| `checkout.html`                 | Product, snapshotted price, confirmation, success and failure states.                                                                              |
| `merchant.html`                 | Onboarding, test/live keys (shown once), products, product editor, webhook config with verification snippets, payout settings, registered domains. |
| `subscriptions.html`            | Mandates with status, next charge, max amount, cancel action.                                                                                      |
| `admin.html`                    | The admin panel.                                                                                                                                   |
| `rates.html`                    | Current and scheduled fees, declared limits.                                                                                                       |
| `reserves.html` (`/reserves`) | FIN outstanding, treasury USDT, reserve ratio, last reconciliation, daily chain head, stated falsifier.                                            |
| `corrections.html`              | Public correction log.                                                                                                                             |
| `security.html`                 | Security posture (links to `/reserves`).                                                                                                         |
| `receipt.html`                  | Printable receipt.                                                                                                                                 |
| `terms.html`, `privacy.html`  | Legal pages, including "not a bank, not insured, redemption depends on reserves at Binance".                                                       |

Every page loads the shared `assets/app.js`, which provides:

- an API client (same-site base URL from a config tag, `Request-Id` surfacing, error-envelope handling),
- auth (access token in memory, refresh via cookie with `X-Fynvar-Client: web`),
- BigInt money formatting via `Intl.NumberFormat`,
- toasts,
- idempotency key generation.

Every page:

- uses a strict CSP, with no third-party or ad scripts,
- follows the design system,
- validates forms client-side but never trusts the client for money amounts.

### R. Design system

Implement the Tailwind config with the palette and animations from Part 1 §1.12. That includes:

- ink text on teal buttons; white on violet only for large or bold text,
- `tabular-nums` on amounts,
- a true minus sign in ruby for negatives and a teal "+" for incoming amounts,
- `prefers-reduced-motion` support,
- skeleton shimmer, pulse-ring, slide-in toasts and the 0.98 press scale,
- designed states for pending, held, settled, failed and reversed, with a timeline view,
- downloadable receipts.

### S. Security and hardening

Implement:

- same-site topology,
- the CSRF model in §2.5,
- CORS with an origin allowlist,
- HTTPS and HSTS,
- strict CSP and `frame-ancestors 'none'` on checkout,
- rate limiting through a bucket table, with auth limits bound to the Argon2 memory budget,
- the payout destination allowlist with cooldown and receiver validation,
- 2FA per policy plus step-up,
- mandatory passkeys for admins,
- maker–checker,
- the audit log,
- the SSRF guard,
- append-only triggers, the hash chain and reconciliation,
- no cookies on `.metax.academy`.

### T. Testing

Write tests (PHPUnit or a simple custom runner) for every item in Part 2 §2.10. That includes:

- property-based I1–I6 tests,
- a 50-process double-spend test,
- webhook chaos (including the ms/s timestamp mix-up),
- payout error-code mapping (exactly one outcome),
- golden fee CSVs (including the ₣1 transfer with a 0.25% fee),
- cross-field settings rules,
- append-only triggers,
- hash-chain tamper detection,
- idempotency 409/422/replay,
- the restore drill.

### U. Deliverables

1. All migration SQL files.
2. All PHP modules, the settings registry and route handlers.
3. All static HTML, JS and CSS files (Tailwind prebuilt).
4. All cron scripts.
5. All tests.
6. `README.md`: environment variables, database initialization (live and test files), Binance sandbox configuration, cron schedule, and deployment to `pay.metax.academy` with the `/api/` proxy to `gloulou.net/fynvar-api/`.
7. `.env.example`: every required variable, including root encryption key, key ID, API-key pepper, Ed25519 signing key, Binance credentials, SMTP and S3.
8. `DEPLOY.md`: Nginx/Apache config (reverse proxy, 308 alias, CSP, HSTS), HTTPS, backups and restore drill, and the **Phase 0 completion record**.

### V. Build order

Follow this order and do not skip ahead:

0. **Phase 0 gate.** Record the legal opinion, Binance written approval, ToS and refund policy, and same-site topology in `DEPLOY.md`. You may build Phases 1–4 against stubs of this record, but **do not start Phase 5 until Phase 0 is complete.**
1. Migrations (STRICT, triggers), settings registry and cache, money helpers, ledger with hash chain, accounts, audit log, idempotency.
2. Auth: rotation and families, TOTP and backup codes, passkeys.
3. Wallet read endpoints.
4. Transfers.
5. Dashboard and login pages.
6. Binance Pay deposits: signing, cert cache, webhook, re-query, processor fees.
7. Payout destinations, withdrawals (hold/settle/release), approval queue, payout polling.
8. Merchant platform (test/live keys), products, checkout, split payments.
9. Subscriptions (mandates), refunds, rolling reserve, cron jobs.
10. Admin panel with all 21 groups, maker–checker, effective-dated fees.
11. Merchant webhook dispatcher (SSRF-safe) and retry cron.
12. Reconciliation, chain anchoring, `/reserves`, correction log, backups and restore drill, rate limiting.
13. Tests (written alongside each module; complete the suite here).
14. README, `.env.example`, DEPLOY.

### W. Constraints and style

- **No framework bloat.** Vanilla PHP, vanilla JS and prebuilt Tailwind. Composer is allowed only for PHPMailer and optionally an S3 client; prefer hand-rolled code where reasonable.
- **No floats for money** in PHP, JS or SQL. Use BigInt on the client.
- **No client-trusted amounts** for credits or prices.
- **No plaintext secrets** in the repo or the database.
- **No skipping idempotency.**
- **Comment every non-obvious financial operation** with its entry structure.
- **Name things clearly:** `postJournal`, not `pj`; `applyBps`, not `fee`.
- **Fail loudly:**
  - Unbalanced journal → throw.
  - Bad webhook signature → 401.
  - Missing setting → registry default (never a hardcoded fallback).
  - Invariant breach → alert and auto-pause.

Begin with the migrations, then the ledger module and its tests, then work outward. After each phase, summarize what was built, what remains, and any assumptions you made. Each phase ends with passing tests.

Do not ask clarifying questions unless a decision genuinely can't be resolved from this spec. Otherwise, make a reasonable choice, document it, and continue.

The spec is complete. Build it.

---

## Appendix A — v1.0 → v1.1 Change Log

### A.1 Critical fixes

| #  | Issue in v1.0                                                                                          | Why it breaks                                                                                                                | v1.1 resolution                                                                                    | Where applied         |
| -- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | --------------------- |
| 1  | API on `gloulou.net`, frontend on `pay.metax.academy`, refresh token in a `SameSite=None` cookie | Cross-site third-party cookies are blocked by Safari ITP, Firefox and Chrome. Double-submit CSRF is impossible across sites. | Same-site API (proxy or subdomain).`__Host-`, `SameSite=Strict` refresh cookie.                | §1.3, §1.9, §2.5   |
| 2  | "Lock the sender's account row"                                                                        | SQLite has no row locks or `FOR UPDATE`.                                                                                   | `BEGIN IMMEDIATE` + guarded `UPDATE … WHERE balance >= amt` + BUSY retry.                     | §1.6, §2.3          |
| 3  | Withdrawal to a "payout address"                                                                       | Binance Payout sends only to BINANCE_ID, EMAIL or OPEN_ID.                                                                   | "Payout destination" (`receiveType` + `receiver`), validated via Validate Receiver.            | §1.8, §2.4, Group 4 |
| 4  | FIN burned at approval                                                                                 | Payouts are asynchronous; a failed payout would lose user FIN and distort reserves.                                          | Two-phase HOLD → SETTLE / RELEASE.                                                                | §2.1.3               |
| 5  | Webhook checked only against the minimum deposit                                                       | Wrong currency or amount would be credited.                                                                                  | Exact currency and amount match, plus Query Order re-check.                                        | §1.8, §2.4          |
| 6  | "Backed 1:1" ignored processor fees                                                                    | The reserve ratio erodes with every deposit.                                                                                 | `system:processor_fees`; deposit fee pricing or operator top-up; reconciliation detects the gap. | §1.7, §2.1          |
| 7  | Contradictory default limits                                                                           | Max withdrawal above the daily cap; auto-approve threshold below the minimum; Binance 80K cap ignored.                       | Cross-field validation; corrected defaults.                                                        | §2.9, Groups 3–4    |
| 8  | Peg mode "dynamic"                                                                                     | Incompatible with 1:1 redeemability; revalues balances.                                                                      | `dynamic` removed; peg and decimals immutable after launch.                                      | §1.7, Group 1        |
| 9  | Minted referral rewards                                                                                | Creates unbacked FIN.                                                                                                        | Funded from `fee_revenue_fin`; rejected if insufficient.                                         | §1.7, Group 17       |
| 10 | Argon2 parallelism 4                                                                                   | Under libsodium threads is effectively 1; RAM exhaustion on shared hosting.                                                  | Detect the backend at boot; tie rate limits to the memory budget.                                  | §1.9, Group 13       |
| 11 | Replay window in seconds vs Binance timestamps in ms                                                   | Rejects every webhook, or accepts all.                                                                                       | `abs(now_ms − ts_ms) ≤ window_s × 1000`.                                                      | §2.4, Group 16       |
| 12 | Order expiry 5–1440 min                                                                               | Binance is believed to cap it at 1 hour (verify in docs).                                                                    | 5–60 min unless the docs show otherwise.                                                          | §2.4, Group 5        |
| 13 | `datetime('now')` called ISO-8601                                                                    | No `T`, `Z` or milliseconds.                                                                                             | `strftime('%Y-%m-%dT%H:%M:%fZ','now')`.                                                          | §1.5                 |
| 14 | Three spellings (Fynvar / Finvar / fynval)                                                             | Hurts support, log searches and trust.                                                                                       | Fynvar / Finvar /`fynvar-api`; `fynval-api` as a 308 alias; ₣ decorative only.                | §1.2, §1.13         |

### A.2 Other v1.1 changes

| Area        | Change                                                                                                                                                                         | Tag                   |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| Ledger      | Explicit sign convention, full chart of accounts, journal catalog, invariants I1–I6                                                                                           | FIX                   |
| Ledger      | STRICT tables, append-only triggers, SHA-256 hash chain, daily anchor                                                                                                          | HARDEN / METAX        |
| Binance     | Exact header and signature spec, certificate cache by SN, nonce replay store,`returnCode` reply body, double JSON decode, Batch Payout fields, error-code mapping            | FIX / PREMIUM         |
| Identity    | Refresh rotation with token families, Ed25519 access tokens, Origin + custom-header CSRF model                                                                                 | FIX                   |
| 2FA         | TOTP replay protection, backup codes, mandatory admin passkeys, SMS removed, step-up for money actions                                                                         | HARDEN                |
| Handles     | Regex, reserved list, 90-day quarantine, rate-limited lookup                                                                                                                   | HARDEN                |
| Admin       | IP allowlist, 15-minute sessions, maker–checker, break-glass account                                                                                                          | HARDEN                |
| Idempotency | Scoping, fingerprints, stored responses, 409/422, permanent ledger-level keys                                                                                                  | FIX / PREMIUM         |
| API         | Prefixed UUIDv7 IDs, amounts as strings + BigInt, error envelope, Request-Id, dated versioning, test/live mode, peppered API keys with scopes,`Fynvar-Signature`, SSRF guard | PREMIUM / HARDEN      |
| Commerce    | Price snapshotting, optional buyer handle, registered redirect domains, clickjacking protection, single-pay sessions, mandates, rolling reserve, KYB, split payments           | FIX / PREMIUM / METAX |
| Settings    | Code registry with DB overrides, versioned cache, key-ID encrypted secrets, effective-dated fees                                                                               | FIX                   |
| Catalog     | Duplicate fields removed (merchant fee, refund fee, webhook retries, per-module enable switches); confirmations removed; new Group 21 Disclosures                              | FIX                   |
| Compliance  | E-money analysis (EMD, MiCA, US MSB, Travel Rule), Binance written approval, safeguarding and wind-down plan, disclosures, Phase 0 gate                                        | FIX                   |
| MetaX       | `/reserves` with falsifier, declared limits, public correction log, merchant #1 use cases, origin isolation, ML-2.3 licensing decision                                       | METAX                 |
| Design      | WCAG contrast fixes, tabular numerals, reduced motion, trust micro-copy, state design, receipts                                                                                | PREMIUM               |
| Testing     | Property-based tests, real 50-process concurrency, webhook chaos, payout path tests, golden fee tables, settings rules, hash-chain tamper test, restore drill                  | HARDEN                |
| Build       | Phase 0 added; exit criteria extended for every phase                                                                                                                          | FIX                   |

### A.3 Notes added during consolidation

These items were not stated explicitly in either source document. They were added to make the merged specification internally consistent, and the developer should confirm or adjust them:

- The `system:merchant_reserve` account, used to implement the rolling reserve.
- Group 4: "Dual approval above" threshold and "Payout status poll interval".
- Group 6: lookup rate limit. Group 7: split-payment toggle and max application fee.
- Group 10: insufficient-balance policy. Group 12: KYB and Travel Rule toggles.
- Group 13: token rotation and backup code fields. Group 20: restore drill schedule.
- Group 17: "Monthly referral budget" default (placeholder).
- Group 2: "Checkout fee payer" options and default (illustrative of the per-fee-type rule).
- Additional cron scripts, tables and pages: `poll-payouts`, `anchor-chain-head`, `apply-scheduled-fees`, `release-rolling-reserve`, `send-renewal-reminders`; `reserves.html`, `corrections.html`, `receipt.html`. These are required by the features above.

---

## Appendix B — Top Risks and Highest-Value Additions

**The three changes most likely to save real money or prevent a production incident:**

1. **#1 Cross-site cookies.** Without a same-site API, token refresh silently fails for a large share of users, and the CSRF design cannot work.
2. **#4 Burning FIN before the payout succeeds.** Asynchronous payout failures would destroy user balances and corrupt reserve figures. The two-phase hold/settle/release design fixes this.
3. **#6 Processor fees eroding the 1:1 backing.** Every deposit would quietly lower the reserve ratio. A dedicated expense account and reconciliation make the gap visible and correctable.

**The highest-value addition** is the falsifiable `/reserves` page. It publishes live reserve figures, the daily chain-head hash and an explicit falsifier, which applies MetaX's own standard of truthfulness and falsifiability directly to Fynvar.
