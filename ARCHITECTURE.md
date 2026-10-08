# SakhiLedger: Architecture

Living design document for the team. Anything marked **🟡 Decision** is open for discussion; anything marked **🔍 Verify** needs checking against Drunix or NPCI documentation before we build on it.

Back to [README](./README.md)

---

## Contents

1. [Goals and non-goals](#1-goals-and-non-goals)
2. [System context](#2-system-context)
3. [Component architecture](#3-component-architecture)
4. [Drunix network design](#4-drunix-network-design)
5. [Data model](#5-data-model)
6. [Chaincode API](#6-chaincode-api)
7. [Key flows](#7-key-flows)
8. [Services](#8-services)
9. [Security and privacy](#9-security-and-privacy)
10. [Offline and accessibility design](#10-offline-and-accessibility-design)
11. [Deployment and environments](#11-deployment-and-environments)
12. [Testing and definition of done](#12-testing-and-definition-of-done)
13. [Proposed work split](#13-proposed-work-split)
14. [Open decisions](#14-open-decisions)
15. [Risks and items to verify](#15-risks-and-items-to-verify)

---

## 1. Goals and non-goals

### Goals

| # | Goal | How we'll show it in the demo |
|---|---|---|
| G1 | Tamper-evident, member-attested SHG records shared by federation, bank and SRLM | Block explorer + tamper-detection alert |
| G2 | Individual, explainable credit score per member | Sakhi Score with 3 reason codes in Tamil/Hindi |
| G3 | Portable, consent-based credit passport | Grant → bank sees passport; revoke → bank loses access |
| G4 | Risk-based Credit Line on UPI instead of a flat ₹5,000 | Limit moves ₹5,000 → ₹15,000 with reasons |
| G5 | Usable by low-literacy, low-smartphone users | Voice query, shared device, offline entry |
| G6 | DPDP-ready privacy by design | No PII on shared ledger, purge on erasure |

### Non-goals (for the hackathon)

- Being a lender or holding funds. **The bank lends; we are a technology service provider.**
- Production Aadhaar authentication (needs AUA/KUA licence). OTP is mocked.
- Replacing LokOS. We import from it (mock CSV for now).
- Real credit bureau integration. We show the export format only.

---

## 2. System context

```mermaid
flowchart TB
    subgraph People
        M[SHG member<br/>'Didi']
        OB[Office-bearers<br/>President / Treasurer]
        BS[Bank Sakhi<br/>assisted mode]
        CO[Bank credit officer]
        SR[SRLM / block staff]
    end

    subgraph SakhiLedger
        APP[Member PWA]
        CON[Bank & SRLM console]
        CORE[Gateway + services]
        L[(Drunix ledger)]
    end

    subgraph External
        UPI[NPCI UPI rails<br/>Credit Line, AutoPay,<br/>Reserve Pay, AePS]
        BH[Bhashini<br/>voice]
        LK[LokOS<br/>import]
        AA[Account Aggregator<br/>optional]
    end

    M --> APP
    OB --> APP
    BS --> APP
    CO --> CON
    SR --> CON
    APP --> CORE
    CON --> CORE
    CORE --> L
    CORE <--> UPI
    CORE --> BH
    LK --> CORE
    AA --> CORE
```

| Actor | What they do |
|---|---|
| **Member (Didi)** | Records savings/repayments, co-signs, views her score, grants or revokes consent, spends and repays via UPI |
| **Office-bearers** | Co-sign cash entries, run the meeting |
| **Bank Sakhi** | Assists members without smartphones; handles cash deposits via AePS |
| **Credit officer** | Verifies passports, sets credit limits, sees repayment events |
| **SRLM / block staff** | Monitors Panchasutra scores and community fund audit trail |

---

## 3. Component architecture

```mermaid
flowchart TB
    subgraph Access["Access layer"]
        PWA["Member PWA<br/>React + Vite<br/>offline-first, voice"]
        BSA["Bank Sakhi mode<br/>(same PWA, assisted role)"]
        BC["Bank console<br/>React"]
        SC["SRLM dashboard<br/>React"]
    end

    subgraph Gateway["API gateway (Node/TS)"]
        AUTH[Auth<br/>device keys + OTP]
        CONS[Consent service]
        API[REST API<br/>OpenAPI 3]
        EVT[Ledger event listener]
        FGW[Fabric Gateway client]
    end

    subgraph Services
        SCORE["Scoring service<br/>Python FastAPI<br/>XGBoost + SHAP"]
        CRED["Credential service<br/>W3C VC issue/verify"]
        UPIA["UPI adapter<br/>mock | sandbox | npci"]
        VOICE["Voice service<br/>Bhashini"]
    end

    subgraph Data["Off-chain data"]
        PG[(PostgreSQL)]
        RD[(Redis)]
        VLT[(Vault)]
    end

    subgraph Ledger["Drunix network"]
        P1[Federation peers]
        P2[Bank peers]
        P3[SRLM peer]
        ORD[Raft orderer]
        CC[[Chaincode: sakhi]]
    end

    PWA --> API
    BSA --> API
    BC --> API
    SC --> API
    API --> AUTH
    API --> CONS
    API --> FGW
    API --> SCORE
    API --> CRED
    API --> UPIA
    API --> VOICE
    FGW --> P1
    FGW --> P2
    FGW --> P3
    P1 --- ORD
    P2 --- ORD
    P3 --- ORD
    P1 --- CC
    P2 --- CC
    P3 --- CC
    EVT --> FGW
    EVT --> PG
    API --> PG
    API --> RD
    AUTH --> VLT
    CRED --> VLT
    SCORE --> PG
```

### Layer responsibilities

| Layer | Responsibility | Owner (proposed) |
|---|---|---|
| Access | UI, offline queue, device key generation, voice capture | Frontend lead |
| Gateway | AuthN/Z, consent enforcement, orchestration, Fabric submit/evaluate, event fan-out | Backend lead |
| Scoring | Feature build from ledger history, score, limit recommendation, reason codes | AI/ML lead |
| Credential | Issue/verify Didi Credit Passport, anchor hash on chain | Backend lead + Security lead |
| UPI adapter | One interface over mock / sandbox / NPCI rails | Backend lead |
| Ledger | Network, chaincode, endorsement, private data | Blockchain lead |
| Security | Keys, threat model, DPDP mapping, pen-test | Security lead |

---

## 4. Drunix network design

### 4.1 Organisations and nodes

| Org (MSP ID) | Represents | Nodes (prototype) | Why it's on the network |
|---|---|---|---|
| `FederationMSP` | SHG federation (VO/CLF), acting for members | 1 peer | Writes meeting records, holds member credit data |
| `BankMSP` | Lending bank | 1 peer | Verifies UPI payments, writes credit-line events, co-endorses credit records |
| `SRLMMSP` | State Rural Livelihood Mission | 1 peer | Oversight, audit of community funds, reads Panchasutra scores |
| `OrdererMSP` | Ordering service | 1 Raft orderer (3 in pilot) | Block ordering |

Each org runs its own Fabric CA. Members do **not** run nodes. Their signatures are verified inside chaincode (see 4.4).

### 4.2 Channels

| Channel | Members | Scope |
|---|---|---|
| `clf-<id>` (prototype: `clf-vellore-01`) | Federation, Bank, SRLM | One channel per cluster-level federation. Keeps data partitioned and lets us scale horizontally. |

🟡 **Decision D1:** one channel per CLF vs one per state. Per-CLF is cleaner for privacy; per-state is fewer channels to manage.

### 4.3 Chaincode

- Single chaincode `sakhi`, written in **Go**.
- Modules (Go packages): `ledger`, `attest`, `panchasutra`, `passport`, `consent`, `vouch`, `credit`.
- Every write takes an **idempotency key** (`clientTxnId`) so offline replays don't double-post.

### 4.4 Endorsement and attestation

There are two layers of "who must agree".

**Org-level endorsement (Fabric policy)**

| Data | Policy | Mechanism |
|---|---|---|
| Default for chaincode | `MAJORITY` of Federation, Bank, SRLM | Chaincode definition |
| Credit-line and passport keys (`CREDIT~*`, `PASSPORT~*`) | `AND('FederationMSP.peer','BankMSP.peer')` | **State-based endorsement** (key-level policy) set on key creation |
| UPI-verified transactions | Must include `BankMSP` endorsement | Key-level policy on `TXN~*` where `mode=UPI` |

**Member-level attestation (inside chaincode)**

| Entry type | Required signatures |
|---|---|
| UPI savings / repayment | Bank endorsement + valid payment reference. No member quorum needed. |
| Cash savings / repayment | Member + 2 office-bearers |
| Internal loan ≤ ₹5,000 | Borrower + 2 office-bearers |
| Internal loan > ₹5,000 | Borrower + majority of members present at the meeting |
| Vouch | Voucher's own signature |
| Consent grant / revoke | Member's own signature |

- Members register an **Ed25519 public key** at enrollment (device-bound, or custodial for shared-device users; see 9.2).
- The client signs `SHA-256(canonical JSON of the entry)`. Chaincode verifies with Go's `crypto/ed25519`.
- Thresholds (₹5,000, majority) live in an on-chain `CONFIG~shg~<id>` key so each SHG can match its own bylaws.

### 4.5 Private data collections

| Collection | Member orgs | Contents | Notes |
|---|---|---|---|
| `memberPII` | Federation, SRLM | Name, phone, KYC reference token (never raw Aadhaar), language | `memberOnlyRead: true`. Purge on erasure request. |
| `memberCredit` | Federation | Per-member amounts, loan schedules, DPD | Input to scoring. Bank never reads this directly; it gets the **passport** instead. |
| `bankCredit` | Federation, Bank | Credit-line sanction, utilisation, repayment events | Feeds the credit ladder |

Fabric automatically stores a hash of each private write on the channel, so tampering is still detectable by every org.

🔍 **Verify V1:** Drunix keeps Fabric 2.5 private data collections and `PurgePrivateData`.

### 4.6 State database

- Drunix offers a **SQL state database**. Dashboards (Panchasutra, fund audit) should query it directly instead of replaying history.
- 🔍 **Verify V2:** query interface and whether rich queries work from Go chaincode.

---

## 5. Data model

### 5.1 On-chain public state (channel world state)

All IDs are pseudonymous. `memberRef = HMAC-SHA256(federationSecret, memberUUID)`. No names, phone numbers or KYC values.

| Key pattern | Value (fields) |
|---|---|
| `SHG~<shgId>` | `clfId, voId, bankBranchCode, formedOn, status, configHash` |
| `MEMBER~<shgId>~<memberRef>` | `role (member/president/treasurer/secretary), pubKey, joinedOn, status` |
| `MEETING~<shgId>~<meetingNo>` | `date, attendeeRefs[], openedBy, closedAt, summaryHash` |
| `TXN~<shgId>~<txnId>` | `meetingNo, memberRef, type, mode, status, attestationCount, privateHash, clientTxnId, createdAt` |
| `PANCHASUTRA~<shgId>~<yyyymm>` | `meetings, savings, interLoaning, repayment, books (each 0–20), total, grade` |
| `PASSPORT~<memberRef>` | `version, vcHash, scoreBand, issuer, issuedAt, expiresAt` |
| `CONSENT~<consentId>` | `memberRef, granteeMSP, purpose, scope[], validUntil, status, signature` |
| `VOUCH~<voucherRef>~<voucheeRef>` | `capINR, expiresAt, status` |
| `CREDIT~<memberRef>` | `bankMSP, sanctionedINR, status, lastEventAt` |
| `CONFIG~shg~<shgId>` | `cashQuorum, loanMajorityThresholdINR, ...` |

**Enums**

- `type`: `SAVING | LOAN_DISBURSE | LOAN_REPAY | INTEREST | FINE | GRANT_IN | BANK_LOAN_IN | BANK_REPAY_OUT`
- `mode`: `UPI | CASH | AEPS`
- `status`: `PENDING_ATTEST | ATTESTED | REJECTED`

### 5.2 Private state

| Collection | Key | Value |
|---|---|---|
| `memberPII` | `PII~<memberRef>` | `name, phoneE164, kycTokenRef, preferredLang, consentToContact` |
| `memberCredit` | `AMT~<txnId>` | `amountPaise, loanId?, dueDate?, interestRateBps?, upiRef?` |
| `memberCredit` | `LOAN~<loanId>` | `principalPaise, schedule[], outstandingPaise, dpd` |
| `bankCredit` | `CL~<memberRef>~<eventId>` | `event (SANCTION/SPEND/REPAY/LIMIT_CHANGE), amountPaise, upiRef, at` |

Money is always stored in **paise (integer)**. No floats.

### 5.3 Off-chain (PostgreSQL)

| Table | Purpose | Ledger link |
|---|---|---|
| `users` | App accounts (member, office-bearer, Bank Sakhi, officer, SRLM) | `memberRef` |
| `devices` | Device public keys, last seen | `MEMBER~` pubKey |
| `sync_queue` | Offline entries awaiting submit | `clientTxnId` |
| `txn_view` | Read model for fast UI | `ledgerTxId`, `rowHash` |
| `scoring_runs` | Feature snapshot, score, reasons, model version | `PASSPORT~` version |
| `consent_view` | Read model for consent screens | `CONSENT~` |
| `audit_log` | Append-only, hash-chained admin actions | — |

Every row that mirrors ledger data stores `ledgerTxId` and `rowHash`. A verifier job recomputes `rowHash` and compares with the ledger. This is how the **tamper demo** works.

---

## 6. Chaincode API

Signatures are indicative; the blockchain lead owns the final version in `chaincode/sakhi/README.md`.

| Function | Who calls | Writes | Checks |
|---|---|---|---|
| `RegisterSHG(shg)` | Federation | `SHG~`, `CONFIG~` | Federation identity |
| `RegisterMember(shgId, memberRef, role, pubKey)` + PII (transient) | Federation | `MEMBER~`, `memberPII` | Unique `memberRef` |
| `OpenMeeting(shgId, meetingNo, date)` | Federation | `MEETING~` | Sequential meeting number |
| `RecordTxn(entry, attestations[])` + amounts (transient) | Federation | `TXN~`, `memberCredit` | Idempotency, quorum rules, signatures |
| `AttestTxn(txnId, attestation)` | Federation | `TXN~` (count/status) | Signer is eligible, no duplicates |
| `VerifyUpiTxn(txnId, upiRef)` | Bank | `TXN~` status → `ATTESTED` | Bank endorsement |
| `CloseMeeting(shgId, meetingNo, summaryHash)` | Federation | `MEETING~` | All txns resolved |
| `ComputePanchasutra(shgId, yyyymm)` | Federation | `PANCHASUTRA~` | Deterministic from ledger |
| `AnchorPassport(memberRef, vcHash, scoreBand, expiresAt)` | Federation | `PASSPORT~` | Endorsed by Federation + Bank |
| `GrantConsent(consent, memberSig)` | Federation (for member) | `CONSENT~` | Member signature |
| `RevokeConsent(consentId, memberSig)` | Federation (for member) | `CONSENT~` status | Member signature |
| `IsConsentValid(consentId, granteeMSP, purpose)` | Any | — (read) | Status, expiry, scope |
| `AddVouch(voucherRef, voucheeRef, capINR, sig)` | Federation | `VOUCH~` | Caps per voucher |
| `RecordCreditEvent(memberRef, event)` + details (transient) | Bank | `CREDIT~`, `bankCredit` | Valid consent exists |
| `GetMemberHistory(memberRef)` | Federation | — (read) | Caller org |

**Events emitted** (consumed by the gateway listener): `TxnRecorded`, `TxnAttested`, `MeetingClosed`, `PanchasutraUpdated`, `PassportAnchored`, `ConsentChanged`, `CreditEventRecorded`.

Private inputs (PII, amounts) are passed as **transient data**, never as plain arguments, so they don't land in the public block.

---

## 7. Key flows

### 7.1 Cash repayment with quorum attestation

```mermaid
sequenceDiagram
    autonumber
    participant T as Shared SHG tablet (PWA)
    participant P as Member + 2 office-bearer phones
    participant G as Gateway
    participant L as Drunix (chaincode)
    participant E as Event listener

    T->>G: POST /txns {type: LOAN_REPAY, mode: CASH, amount, clientTxnId}
    G->>L: RecordTxn (status PENDING_ATTEST, amount via transient)
    L-->>G: txnId
    G-->>P: Push "Approve ₹500 repayment by Lakshmi?"
    P->>P: Sign SHA-256(entry) with device key
    P->>G: POST /txns/{id}/attest {signature}
    G->>L: AttestTxn (x3)
    L->>L: Verify Ed25519 signatures, count = 3 → ATTESTED
    L-->>E: TxnAttested event
    E->>G: Update txn_view + rowHash
```

### 7.2 UPI savings, auto-verified

```mermaid
sequenceDiagram
    autonumber
    participant M as Member UPI app
    participant U as UPI adapter
    participant G as Gateway
    participant L as Drunix (chaincode)

    M->>U: Pay ₹100 to SHG collect/QR (payment intent with clientTxnId)
    U-->>G: Webhook: SUCCESS, upiRef
    G->>G: Verify webhook signature, match clientTxnId
    G->>L: RecordTxn (mode UPI) via Federation
    G->>L: VerifyUpiTxn(upiRef) via Bank identity
    L->>L: Key-level policy requires BankMSP endorsement
    L-->>G: ATTESTED
```

### 7.3 Passport, consent and credit line

```mermaid
sequenceDiagram
    autonumber
    participant M as Member (PWA)
    participant G as Gateway
    participant S as Scoring service
    participant C as Credential service
    participant L as Drunix
    participant B as Bank console
    participant U as UPI adapter

    G->>L: GetMemberHistory(memberRef)
    G->>S: POST /score {features}
    S-->>G: score, band, recommendedLimit, reasons[3]
    G->>C: Issue DidiCreditPassport VC
    C-->>G: signed VC (JWT)
    G->>L: AnchorPassport(vcHash) [Federation + Bank endorse]
    M->>G: Grant consent to Bank X, purpose CREDIT_LINE, 90 days (signed)
    G->>L: GrantConsent
    B->>G: GET /bank/passports/{memberRef}
    G->>L: IsConsentValid?
    G-->>B: VC + reasons
    B->>B: Verify VC signature + compare hash with ledger
    B->>G: POST /bank/credit-lines {limit: 15000}
    G->>M: "Bank proposes ₹15,000. Accept?" (explicit yes)
    M->>G: Accept (signed)
    G->>U: creditLine.sanction(memberRef, 15000)
    G->>L: RecordCreditEvent(SANCTION)
```

### 7.4 Repayment loop (credit ladder)

```mermaid
sequenceDiagram
    autonumber
    participant U as UPI adapter (AutoPay)
    participant G as Gateway
    participant L as Drunix
    participant S as Scoring

    U-->>G: AutoPay debit SUCCESS (repayment)
    G->>L: RecordCreditEvent(REPAY) [Bank]
    L-->>G: CreditEventRecorded
    G->>S: Re-score member (nightly or on event)
    S-->>G: New band / limit recommendation
    G->>G: If band improved, notify member + bank (no auto-increase)
```

### 7.5 Tamper detection

1. A verifier job (cron + on-read) loads `txn_view` rows.
2. It recomputes `rowHash` and fetches the ledger record by `ledgerTxId`.
3. On mismatch it raises a `TAMPER_SUSPECTED` alert on the SRLM dashboard and bank console, with row, ledger value and timestamp.

---

## 8. Services

### 8.1 Gateway REST API (v1)

| Method | Path | Caller | Purpose |
|---|---|---|---|
| POST | `/auth/otp/request`, `/auth/otp/verify` | All | Login (OTP mocked) |
| POST | `/devices` | Member | Register device public key |
| POST | `/shgs`, `/shgs/{id}/members` | Federation admin | Onboarding / LokOS import |
| POST | `/meetings`, `/meetings/{id}/close` | Office-bearer | Meeting lifecycle |
| POST | `/txns` | Office-bearer / Bank Sakhi | Record entry (supports batch for offline sync) |
| POST | `/txns/{id}/attest` | Member / office-bearer | Co-sign |
| GET | `/members/{ref}/score` | Member | Own score + reasons |
| GET | `/members/{ref}/passport` | Member | Own passport |
| POST | `/consents`, `/consents/{id}/revoke` | Member | Consent |
| POST | `/vouches` | Member | Vouch for a peer |
| GET | `/bank/passports/{ref}` | Bank officer | Consent-gated passport view |
| POST | `/bank/credit-lines` | Bank officer | Propose limit |
| POST | `/credit-lines/{id}/accept` | Member | Explicit acceptance |
| GET | `/srlm/panchasutra?clf=` | SRLM | Group scores |
| GET | `/alerts` | Bank / SRLM | Tamper and anomaly alerts |
| POST | `/webhooks/upi` | UPI adapter | Payment callbacks (signed) |
| POST | `/voice/query` | Member | Audio in → intent → audio out |

Spec lives in `docs/api/openapi.yaml`. **The OpenAPI file is the contract between backend and frontend.**

### 8.2 Scoring service

**Features (per member, rolling 24 months)**

| Group | Features |
|---|---|
| Savings | Regularity (% meetings with savings), streak, total, trend |
| Internal credit | Loans taken, on-time repayment ratio, max days past due, repaid-early count |
| Bank credit | Utilisation, on-time AutoPay ratio (once active) |
| Participation | Attendance rate, tenure, office-bearer role |
| Group | Panchasutra total and grade, group default rate |
| Data quality | Share of UPI-verified vs cash entries, attestation completeness |
| Social | Vouches received (capped), vouches given that went bad |

**Model**

- XGBoost with **monotonic constraints**, so better repayment can never lower the score.
- Trained on synthetic data calibrated to published NABARD/NRLM aggregates. This must be clearly labelled synthetic in the demo.
- Output: score 0–100, band A–E, recommended limit on the **credit ladder** (₹5k → ₹10k → ₹15k → ₹25k → ₹50k).
- **Reason codes:** top 3 SHAP contributors, mapped to plain-language templates in Tamil, Hindi and English (e.g. "38 on-time repayments").

**Fairness rules**

- Never use caste, religion or any protected attribute as a feature.
- Report score distribution by district and SHG age in the SRLM dashboard.

**API**

- `POST /score` → `{score, band, recommendedLimitINR, reasons[], modelVersion, featureHash}`
- `GET /health`, `GET /model-card`

### 8.3 Credential service: Didi Credit Passport

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1"],
  "type": ["VerifiableCredential", "DidiCreditPassport"],
  "issuer": "did:web:federation.sakhiledger.example",
  "issuanceDate": "2026-10-08T00:00:00Z",
  "expirationDate": "2027-01-06T00:00:00Z",
  "credentialSubject": {
    "id": "did:key:z6Mk...memberPseudonym",
    "shgRef": "hash",
    "scoreBand": "B",
    "recommendedLimitINR": 15000,
    "panchasutraGrade": "A",
    "reasons": ["R_ONTIME_REPAY_38", "R_SAVINGS_REGULAR_96", "R_GROUP_GRADE_A"],
    "dataWindow": "2024-10..2026-09",
    "ledgerAnchor": { "channel": "clf-vellore-01", "key": "PASSPORT~<memberRef>", "version": 3 }
  }
}
```

- Format: JWT-VC signed with the federation's Ed25519 key, held in Vault.
- Verify = check signature + check `SHA-256(vc) == PASSPORT~.vcHash` on the ledger + check consent is valid.
- 🟡 **Decision D3:** JWT-VC vs SD-JWT (selective disclosure, e.g. show band but hide reasons).

### 8.4 UPI adapter

One interface; the implementation is selected by `UPI_MODE=mock|sandbox|npci`.

```ts
// services/upi-adapter/src/types.ts
export interface UpiAdapter {
  // Collect savings / repayments
  createPaymentIntent(p: { clientTxnId: string; payeeVpa: string; amountPaise: number; note: string }): Promise<{ intentUrl: string; qrPng: string }>;
  verifyWebhook(headers: Record<string, string>, body: string): { ok: boolean; clientTxnId?: string; upiRef?: string; status?: "SUCCESS" | "FAILED" };

  // Credit Line on UPI
  sanctionCreditLine(p: { memberRef: string; limitPaise: number }): Promise<{ creditLineId: string }>;
  payFromCreditLine(p: { creditLineId: string; merchantVpa: string; amountPaise: number }): Promise<{ upiRef: string }>;

  // Repayment and savings automation
  createAutoPayMandate(p: { creditLineId: string; maxAmountPaise: number; frequency: "WEEKLY" | "MONTHLY" }): Promise<{ mandateId: string }>;
  createReserveBlock(p: { memberRef: string; blockPaise: number; validDays: number }): Promise<{ blockId: string }>;

  // Assisted cash
  aepsThirdPartyDeposit(p: { depositorRef: string; beneficiaryAccountRef: string; amountPaise: number }): Promise<{ rrn: string }>;
}
```

- `mock` returns deterministic responses for demos and tests.
- `sandbox` uses a payment-aggregator test mode for real UPI intents.
- `npci` is filled in once hackathon API access is granted.

### 8.5 Voice service

- Bhashini ASR (speech → text) → intent classifier (small rule set: balance, limit, last payment, next due) → template answer → Bhashini TTS.
- Languages: Tamil and Hindi first. 🟡 **Decision D7:** add English or Telugu?
- No raw audio is stored; only the transcript hash is logged.

---

## 9. Security and privacy

### 9.1 Identity

| Who | Identity | Where the key lives |
|---|---|---|
| Org peers / admins | Fabric MSP X.509 | Org CA; private keys never leave the host |
| Gateway (per org) | Fabric client identity per org | Vault |
| Member with own phone | Ed25519 keypair | WebCrypto, non-extractable, in the device |
| Member on shared tablet | Custodial Ed25519 key | Vault, unlocked by member PIN/OTP per action |
| Federation issuer | Ed25519 VC signing key | Vault (transit engine) |

### 9.2 Threat model (STRIDE summary)

| Threat | Example | Mitigation |
|---|---|---|
| **Spoofing** | Someone attests as another member | Device-bound keys, PIN per signature, OTP on new device, 2-of-3 office-bearer quorum |
| **Tampering** | Bookkeeper edits an old entry | Ledger is source of truth; `rowHash` verifier; private data hashes on channel |
| **Repudiation** | "I never approved that loan" | Signed attestations stored on chain |
| **Information disclosure** | Bank reads member data without consent | PII off-chain / PDC; bank only gets passport; consent checked on every read |
| **Denial of service** | Network outage at meeting | Offline queue, idempotent replay, rate limits at gateway |
| **Elevation of privilege** | Bank identity writes federation records | Function-level MSP checks in chaincode; least-privilege roles |
| **Collusion** | Group fakes cash repayments to inflate scores | Cash weighted below UPI; anomaly detection (round amounts, same-minute bursts); limits climb gradually; vouch downside |
| **Model risk** | Opaque or unfair score | Monotonic constraints, reason codes, bank officer makes the final call |
| **Supply chain** | Leaked keys in repo | `drunix/`, `crypto-config/`, `.env` git-ignored; secret scanning in CI |

Full model: `docs/security/threat-model.md` (Security lead).

### 9.3 DPDP Act 2023 / Rules 2025 mapping

| Requirement | Our design |
|---|---|
| Notice and specific consent | Purpose-specific consent screen in the member's language |
| Withdrawal as easy as giving | One-tap revoke, effective on next read |
| Data minimisation | Only pseudonymous refs and hashes on shared ledger |
| Right to erasure | Delete off-chain PII + purge `memberPII` private data |
| Security safeguards | Encryption at rest (Postgres, Vault), mutual TLS, audit log |
| Breach notification | Alert runbook in `docs/security/incident.md` |
| Consent Manager (from 13 Nov 2026) | Consent records are interoperable JSON; can plug into a registered Consent Manager later |

### 9.4 Lending compliance

- The **bank is the lender**. SakhiLedger is a technology service provider.
- Credit limits **never increase without explicit member acceptance**.
- Score is decision support; the **credit officer makes the decision**.

---

## 10. Offline and accessibility design

| Concern | Design |
|---|---|
| No network at meeting | PWA stores signed entries in IndexedDB `sync_queue`; syncs in order when online |
| Duplicate submits | `clientTxnId` idempotency enforced in chaincode |
| Conflicting edits | Entries are append-only; corrections are new reversal entries, never edits |
| Low literacy | Icon-first UI, voice readout of every amount, numbers in local script |
| No smartphone | Shared SHG tablet with per-member PIN; Bank Sakhi assisted mode |
| Feature phone users | Roadmap: IVR + UPI 123PAY for payments |
| Accessibility | WCAG AA contrast, large tap targets, screen-reader labels |

---

## 11. Deployment and environments

### 11.1 Local (docker compose)

| Service | Port | Notes |
|---|---|---|
| Drunix peers ×3, orderer, CAs | per network script | From `npci/drunix` test network |
| `gateway` | 8080 | Node 20 |
| `scoring` | 8090 | Python 3.11 |
| `credential` | 8070 | Can live inside gateway for the prototype |
| `upi-adapter` | 8060 | `UPI_MODE=mock` by default |
| `voice` | 8050 | Needs Bhashini API key |
| `postgres` | 5432 | |
| `redis` | 6379 | |
| `vault` | 8200 | Dev mode locally only |
| `member-pwa` | 5173 | |
| `bank-console` | 5174 | |

### 11.2 Environment variables (`.env.example`)

```
UPI_MODE=mock
FABRIC_CHANNEL=clf-vellore-01
FABRIC_CHAINCODE=sakhi
FEDERATION_GATEWAY_PEER=localhost:7051
BANK_GATEWAY_PEER=localhost:9051
SRLM_GATEWAY_PEER=localhost:11051
DATABASE_URL=postgres://sakhi:sakhi@localhost:5432/sakhi
REDIS_URL=redis://localhost:6379
VAULT_ADDR=http://localhost:8200
BHASHINI_API_KEY=changeme
SCORING_URL=http://localhost:8090
```

### 11.3 Demo environment

- One AWS EC2 instance (or a team laptop) running everything via compose.
- Prometheus + Grafana dashboard showing ledger tx/sec and gateway latency for the "it scales" slide.

### 11.4 CI (GitHub Actions)

- Go: `go vet`, `go test ./...` for chaincode
- Node: lint, type-check, unit tests for gateway and apps
- Python: `ruff`, `pytest` for scoring
- Secret scanning (gitleaks) on every PR
- OpenAPI lint

---

## 12. Testing and definition of done

| Level | What | Owner |
|---|---|---|
| Chaincode unit | Quorum rules, idempotency, consent checks, SBE policies | Blockchain |
| Integration | Gateway ↔ test network round trips | Backend |
| Contract | OpenAPI schema tests (frontend mocks generated from spec) | Backend + Frontend |
| Model | AUC, calibration, monotonicity checks on held-out synthetic data | AI/ML |
| E2E | Playwright run of the full demo script | Frontend |
| Load | k6 against gateway, report tx/sec | Security / Backend |
| Security | Threat model review, OWASP ZAP scan, secret scan | Security |

**Definition of done for any PR**

- Tests pass in CI
- One reviewer from a different role approves
- API or chaincode changes update `openapi.yaml` or `chaincode/sakhi/README.md`
- No secrets, no PII in fixtures (synthetic only)

---

## 13. Proposed work split

> 🟡 **For team discussion.** Roles are proposals, not assignments. Swap freely based on who wants what.

### 13.1 Roles and ownership

| Role | Owns | Primary deliverables | Depends on |
|---|---|---|---|
| **R1 Blockchain lead** | `network/`, `chaincode/sakhi/` | Drunix network script, chaincode with all functions in §6, SBE + PDC config, chaincode README | — |
| **R2 Backend lead** | `gateway/`, `services/upi-adapter/`, `credentials/` | OpenAPI spec, Fabric Gateway integration, event listener, UPI adapter (mock + sandbox), VC issue/verify | R1 function signatures |
| **R3 Frontend lead** | `apps/member-pwa/`, `apps/bank-console/` | Member PWA (offline, voice UI, signing), Bank console, SRLM dashboard | R2 OpenAPI spec |
| **R4 AI/ML lead** | `services/scoring/`, `services/voice/`, `data/synthetic/` | Synthetic data generator, scoring model + SHAP reasons, model card, voice intents | R1 history format |
| **R5 Security & research lead** | `docs/security/`, `docs/research/`, pitch | Threat model, Vault setup, DPDP mapping, pen-test, 2–3 SHG interviews, deck + demo script | Everyone |

### 13.2 Contracts to agree on in week 1

These unblock parallel work. Each has one owner and must be merged before others build on it.

| Contract | File | Owner | Consumers | Due |
|---|---|---|---|---|
| Chaincode function signatures + event payloads | `chaincode/sakhi/README.md` | R1 | R2, R4 | End of week 1 |
| REST API | `docs/api/openapi.yaml` | R2 | R3, R5 | End of week 1 |
| Member history / feature input schema | `services/scoring/schema.json` | R4 | R2 | End of week 1 |
| UPI adapter interface | `services/upi-adapter/src/types.ts` | R2 | R4 (demo), R5 | End of week 1 |
| Synthetic dataset spec | `data/synthetic/README.md` | R4 | R1 (seed), R3 (demo UI) | Mid week 1 |
| Design system + screen list | Figma / Claude Design link in `docs/design.md` | R3 | R5 (deck) | End of week 1 |

### 13.3 Week-by-week plan per role

| Week | R1 Blockchain | R2 Backend | R3 Frontend | R4 AI/ML | R5 Security & research |
|---|---|---|---|---|---|
| **1** | Network up, channel, empty chaincode deployed; signatures doc | OpenAPI draft, gateway skeleton, mock UPI adapter | Wireframes, PWA shell, design system | Synthetic data generator v0, feature schema | Threat model v0, interview guide, 2 SHG interviews booked |
| **2** | `RegisterSHG/Member`, meetings, `RecordTxn`, attestation + quorum | Fabric submit/evaluate, event listener, Postgres read models | Meeting + entry screens, offline queue, device keys | Feature builder from ledger history | Vault setup, key handling review, interviews done |
| **3** | SBE for credit/passport, PDCs, `AnchorPassport`, consent | Credential service, sandbox UPI, webhooks | Score screen, consent flow, Tamil/Hindi strings | Model v1 + SHAP reasons + model card | Pen-test plan, DPDP mapping, deck v1 |
| **4** | Vouch, credit events, Panchasutra, purge | Bank APIs, credit line + AutoPay flows | Bank console, SRLM dashboard, voice UI | Voice intents with Bhashini, re-score on events | ZAP scan, secret scan, demo script v1 |
| **5** | Load test support, bug fixes | Tamper verifier job, load test | E2E Playwright, polish, accessibility pass | Fairness report, calibration | Fix-verify security findings, backup video |
| **6** | Freeze | Freeze | Freeze | Freeze | Dry runs ×3, final deck |

### 13.4 RACI for key deliverables

R = Responsible, A = Accountable, C = Consulted, I = Informed

| Deliverable | R1 | R2 | R3 | R4 | R5 |
|---|---|---|---|---|---|
| Drunix network + chaincode | **R/A** | C | I | C | C |
| OpenAPI + gateway | C | **R/A** | C | C | I |
| Member PWA + consoles | I | C | **R/A** | I | C |
| Scoring + reasons | C | C | I | **R/A** | C |
| Didi Credit Passport | C | **R** | I | C | **A** |
| Threat model + DPDP | C | C | C | C | **R/A** |
| Demo + deck | I | C | C | C | **R/A** |

### 13.5 Working agreements

- **Branches:** `main` is protected; feature branches `r<role>/<short-name>`; squash merge.
- **Issues:** one GitHub issue per task, labelled `chaincode`, `gateway`, `pwa`, `console`, `scoring`, `voice`, `security`, `research`, `pitch`.
- **Board:** GitHub Projects with columns Backlog → This week → In progress → Review → Done.
- **Sync:** 15-minute standup 3×/week; Sunday demo-to-each-other.
- **Decisions:** record in `docs/decisions/NNN-title.md` (one paragraph each).

---

## 14. Open decisions

| ID | Question | Options | Proposed |
|---|---|---|---|
| D1 | Channel granularity | Per CLF / per state | Per CLF |
| D2 | Where scoring runs | Federation-operated / bank-operated / neutral | Federation-operated; bank gets passport only |
| D3 | Credential format | JWT-VC / SD-JWT | JWT-VC for prototype |
| D4 | Mobile app | PWA only / React Native | PWA only |
| D5 | Payment sandbox | Razorpay test / Cashfree test / mock only | Pick one with UPI intent + webhooks |
| D6 | Custodial keys for shared device | Vault custodial / no shared-device mode | Vault custodial with per-action PIN |
| D7 | Languages | Tamil + Hindi / + English / + Telugu | Tamil + Hindi + English |
| D8 | Demo hosting | Laptop only / EC2 + laptop backup | EC2 + laptop backup |
| D9 | LokOS import format | Mock CSV / skip | Mock CSV of SHG + members |
| D10 | Credit ladder steps | Fixed steps / bank-configurable | Bank-configurable, default ₹5k→₹50k |

---

## 15. Risks and items to verify

### Risks

| Risk | Impact | Mitigation |
|---|---|---|
| NPCI API access late or partial | Payment flows can't go live | Adapter with mock + sandbox; label clearly in demo |
| Drunix setup issues on Windows | Lost week 1 | WSL2 + Docker; one owner; fallback to a Linux EC2 |
| Synthetic data looks unconvincing | Judges doubt the score | Calibrate to NABARD/NRLM aggregates; show reasons; add interview quotes |
| Scope creep | Nothing finishes | Must-have path = attest → ledger → passport → consent → credit-line mock |
| Shared-device key custody questioned | Security credibility | Document custodial model honestly; show per-action PIN |
| Overlap with other Track 4 teams | Weak novelty | Lead with SHG + flat ₹5,000 gap in first 30 seconds |

### To verify (🔍)

| ID | Item | Owner |
|---|---|---|
| V1 | Private data collections and `PurgePrivateData` work on Drunix | R1 |
| V2 | SQL state DB query support from Go chaincode | R1 |
| V3 | Fabric Gateway SDK (Node) compatibility with Drunix peers | R2 |
| V4 | Drunix test-network script names and ports | R1 |
| V5 | Hackathon rules on NPCI API sandbox access and timelines | R5 |
| V6 | Bhashini API access and rate limits | R4 |
| V7 | Credit Line on UPI current interchange/limits for SHG pilots | R5 |

---

_Last updated: 8 Oct 2026. Edit via PR; discuss in the linked issue._
