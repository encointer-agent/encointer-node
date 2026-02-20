# Encointer via USSD: A Concept for Feature Phone Access

## Motivation

Encointer's mission is to provide local community currencies with Universal Basic Income (UBI) to everyone — especially the unbanked and citizens of dysfunctional states. Yet the current protocol requires a smartphone app with camera, Bluetooth, GPS, and internet connectivity. This excludes a massive portion of the target population:

- Africa has ~18% internet penetration but over 80% mobile penetration (960M+ mobile users)
- 1.1 billion people worldwide lack state-issued ID documents
- Feature phones (no camera, no GPS, no app store) remain dominant in many regions

USSD (Unstructured Supplementary Service Data) is the proven technology behind M-Pesa and similar services that reach hundreds of millions of unbanked users. This document explores how Encointer's protocol can be adapted to work over USSD while preserving decentralization as much as possible.

## USSD Constraints

Any design must work within these hard limitations:

| Constraint | Value |
|---|---|
| Session duration | 120–180 seconds total |
| Idle timeout | ~30 seconds per interaction |
| Message length | 160–182 characters max |
| Concurrent sessions | 1 |
| Interface | Text-only, numbered menus |
| Data persistence | None (real-time session only) |
| Network | GSM (2G/3G); no internet required |
| Crypto capabilities | None on the phone itself |

Key implication: **the phone cannot hold private keys, sign transactions, or generate ZK proofs**. All cryptographic operations must happen server-side or be replaced with alternative trust mechanisms.

## Architecture Overview

```
┌─────────────┐     GSM/USSD      ┌──────────────────┐     RPC      ┌──────────────┐
│ Feature Phone│◄────────────────►│  USSD Gateway     │◄───────────►│  Encointer    │
│ (dumb phone) │   *384*123#      │  (Operator-hosted │              │  Blockchain   │
└─────────────┘                   │   or independent) │              │  Node         │
                                  └──────┬───────────┘              └──────────────┘
                                         │
                                  ┌──────▼───────────┐
                                  │  Account Manager  │
                                  │  (key custody,    │
                                  │   proof generation,│
                                  │   session state)  │
                                  └──────────────────┘
```

### Components

1. **Feature Phone** — the user's basic GSM phone. Dials a USSD short code. Has no computational capability beyond displaying text menus and sending numeric responses.

2. **USSD Gateway** — bridges USSD sessions to the Account Manager. Can be operated by a telco partner, an NGO, or a community cooperative. Multiple independent gateways can coexist (see Decentralization section).

3. **Account Manager** — the critical new component. Holds cryptographic material on behalf of users, signs transactions, generates ZK proofs for offline payments, and manages session state. This is where the trust trade-off lives.

4. **Encointer Blockchain Node** — unchanged. The on-chain protocol remains exactly as-is.

## Mapping Encointer Features to USSD

### 1. Account Creation & Identity

**Current (smartphone):** The app generates a local keypair. The user controls their own private key.

**USSD approach:** The user dials a short code (e.g., `*384*123#`). The Account Manager generates a keypair associated with the user's phone number (or a PIN-derived identifier). The private key is stored in the Account Manager's secure enclave / HSM.

```
USSD Session: Account Creation
─────────────────────────────
Welcome to Encointer!
1. Create account
2. Check balance
3. Send money
4. Ceremonies

> 1

Enter a 4-digit PIN:
> ****

Confirm PIN:
> ****

Account created!
Your ID: EN-7X3K
Save your recovery code:
MANGO-RIVER-SEVEN

Reply 1 to continue
```

**Authentication model:** Phone number + PIN. The PIN is never stored in cleartext — it is used as input to derive an encryption key for the account's private key at rest. The phone number serves as a routing identifier, not a security factor (SIM swap attacks must be mitigated — see Security section).

**Recovery:** A human-readable recovery phrase (3 words from a wordlist, mapped to a secret seed) allows account recovery if the user changes phone numbers.

### 2. Community Membership & Registration

**Current:** The app shows a map of nearby communities. The user selects one and registers.

**USSD approach:** Communities are identified by short human-readable codes. A user can be told their community code by a neighbor or community leader.

```
USSD Session: Join Community
────────────────────────────
Your communities: (none)

1. Join a community
2. Back

> 1

Enter community code
(ask your neighbor):
> ZURICH01

Join "Zurich LEU"?
1. Yes
2. No

> 1

Joined! Next ceremony:
Mar 2, 10:00 AM
Location assigned 24h before
```

### 3. Ceremony Participation (Proof-of-Personhood)

This is the hardest feature to adapt. The current ceremony requires:
- GPS to verify location
- Camera to scan QR codes
- Bluetooth/internet to exchange attestations
- All within a tight time window

**USSD approach — Hybrid Meetups:**

Feature phone users attend the same physical meetups as smartphone users. The key insight: **smartphone participants can attest feature phone participants**.

#### Flow:

1. **Registration (days before):** The USSD user dials in and registers for the upcoming ceremony. The Account Manager submits the registration extrinsic on their behalf.

```
Ceremony Registration
─────────────────────
Next ceremony: Mar 2
Phase: REGISTERING

1. Register for ceremony
2. Check status
3. Back

> 1

Registered for Mar 2!
Check back 24h before
for your assignment.
```

2. **Assignment (24h before):** The user dials in to learn their meetup location and time.

```
Your Assignment
───────────────
Ceremony: Mar 2
Time: 10:00 AM
Location: Market Square
         (near fountain)
Meetup #: 7
Group size: 3-12 people

Reply 1 to confirm
```

3. **Attendance & Attestation (at the meetup):**

The feature phone user shows up physically. They are identified by a **short attestation code** displayed or read aloud:

```
Your Meetup Code
────────────────
Tell this code to a
smartphone participant:

  >> 4 7 2 9 <<

They will scan/enter it.
Valid for 5 minutes.
Code changes after use.

1. Get new code
2. Back
```

A smartphone participant enters/verifies this code in their app, which:
- Confirms the feature phone user is physically present
- Creates an attestation linking the USSD user's on-chain identity to this meetup
- The Account Manager co-signs the attestation on behalf of the USSD user

**Alternative: Voice-call attestation.** For meetups where ALL participants are on feature phones, a voice-call-based ceremony is possible:
- All assigned participants call a shared conference number at the designated time
- Each participant enters their unique code via DTMF (phone keypad tones)
- The system verifies all codes were entered within the time window from the same approximate cell tower area (cell-ID based location)
- This is weaker than GPS+camera but still provides meaningful location proof

#### Location Verification for Feature Phones

Without GPS, location must be verified through alternative means:

- **Cell-ID triangulation:** The GSM network knows which cell tower the phone is connected to. The USSD gateway can request this information (with appropriate telco partnerships). Accuracy: 100m–several km depending on cell density.
- **Smartphone co-attestation:** At least one smartphone user in the meetup provides GPS coordinates and attests to the feature phone user's presence.
- **Minimum smartphone ratio:** A policy parameter could require at least 1 smartphone participant per meetup to anchor location verification.

### 4. Balance Queries & Demurrage

Straightforward over USSD. Demurrage is computed on-chain; the gateway simply queries and displays the current balance.

```
Your Balance
────────────
Community: Zurich LEU
Balance: 42.7 LEU
(demurrage: -7%/month)

Last income: Feb 20
Next ceremony: Mar 2

1. Send money
2. Transaction history
3. Back
```

### 5. Transfers (Sending Money)

**Current:** Scan QR code or enter address in app.

**USSD approach:** Recipients are identified by phone number (mapped to on-chain accounts via the Account Manager) or by short Encointer IDs.

```
Send Money
──────────
Community: Zurich LEU
Balance: 42.7 LEU

Enter recipient phone
or Encointer ID:
> 0712345678

Amount to send:
> 5

Send 5.0 LEU to
Maria K. (0712345678)?
Enter PIN to confirm:
> ****

Sent! New balance: 37.7 LEU
Recipient notified via SMS.
```

The Account Manager signs and submits the transfer extrinsic. An SMS confirmation is sent to both parties (USSD sessions are ephemeral, so SMS is used for receipts).

### 6. Offline Payments (ZK-Proof Based)

The existing offline payment system uses Groth16 ZK proofs to enable payments without an active blockchain connection. The sender generates a proof that they own funds, and the recipient can later submit this proof for on-chain settlement.

**USSD approach:**

Feature phone users cannot generate ZK proofs locally. Two options:

#### Option A: Gateway-Assisted Proof Generation (Recommended)

The Account Manager generates the ZK proof on behalf of the sender. This requires the Account Manager to have access to the user's `zk_secret` (derived from their keypair, which it already custodies).

```
Offline Payment
───────────────
This creates a payment
code valid without
internet.

Recipient phone/ID:
> 0712345678

Amount:
> 10

Your offline code:
ENP-7F3K-M2X9-QR4T

Give this code to the
recipient. Valid 7 days.
```

The "offline payment code" is a compact encoding of the proof, nullifier, amount, and recipient — short enough to be read aloud, written on paper, or sent via SMS. The recipient (or anyone) later submits it for settlement.

**Encoding scheme:** The full Groth16 proof is ~192 bytes, too large for a human-readable code. Instead:
- The Account Manager stores the full proof server-side, indexed by a short lookup code
- The lookup code (e.g., `ENP-7F3K-M2X9-QR4T`, 16 chars) is given to the user
- At settlement time, the submitter retrieves the full proof via the lookup code
- The on-chain settlement is identical to the smartphone flow

This introduces a liveness requirement on the Account Manager for settlement, but the payment commitment itself (the nullifier) is cryptographically binding regardless.

#### Option B: SMS-Based Proof Relay

The full proof is sent to the recipient via SMS (possibly split across multiple messages). The recipient's Account Manager reassembles and submits it. This removes the liveness dependency but is less user-friendly.

### 7. Bazaar (Marketplace)

**Current:** Browse and post offers in the app.

**USSD approach:** Simplified listing and browsing via text menus.

```
Bazaar - Zurich LEU
────────────────────
1. Browse offers
2. My listings
3. New listing
4. Back

> 1

Offers near you:
1. Eggs 12pc - 3 LEU
   Maria K. 0712345678
2. Haircut - 8 LEU
   James M. 0723456789
3. Next page

> 1

Eggs 12pc - 3 LEU
Maria K.
Call: 0712345678
SMS: "BUY EN-4521"

1. Buy now (3 LEU)
2. Back
```

Character limits mean only short descriptions are feasible. Full marketplace experience remains a smartphone advantage, but basic buy/sell works.

### 8. Democracy & Governance

**Current:** Vote on proposals in the app.

**USSD approach:** Proposals are summarized in 160 characters. Users vote by number.

```
Community Vote
──────────────
Proposal #12:
"Increase UBI from 10
to 12 LEU per ceremony"
By: Council

1. Vote YES
2. Vote NO
3. Abstain
4. More details (SMS)

> 1

Enter PIN to confirm:
> ****

Vote recorded! YES
Results after Mar 5.
```

For complex proposals, a "More details" option sends the full text via SMS (which has no length limit in practice when using concatenated SMS).

### 9. Faucet

Faucet claims can be initiated via USSD. The Account Manager submits the faucet extrinsic.

```
Claim Faucet
────────────
Available faucets:
1. Welcome bonus (1 LEU)
2. Back

> 1

Claimed 1.0 LEU!
New balance: 1.0 LEU
```

## Decentralization Considerations

The introduction of an Account Manager creates a centralization risk. Here are strategies to mitigate this:

### 1. Multiple Independent Gateways

Anyone can run an Account Manager + USSD Gateway pair. Different telcos, NGOs, cooperatives, or community organizations can each operate their own gateway with its own USSD short code.

```
┌──────────┐    *384*100#    ┌──────────────┐
│ User A   │────────────────►│ Gateway A    │──►│
└──────────┘                 │ (NGO-run)    │   │
                             └──────────────┘   │   ┌────────────┐
┌──────────┐    *384*200#    ┌──────────────┐   ├──►│ Encointer  │
│ User B   │────────────────►│ Gateway B    │──►│   │ Blockchain │
└──────────┘                 │ (Coop-run)   │   │   └────────────┘
                             └──────────────┘   │
┌──────────┐    *384*300#    ┌──────────────┐   │
│ User C   │────────────────►│ Gateway C    │──►│
└──────────┘                 │ (Telco-run)  │   │
                             └──────────────┘
```

Users can switch gateways by migrating their account (exporting their recovery phrase and importing it on a different gateway). The on-chain identity remains the same regardless of which gateway is used.

### 2. Account Portability

The Account Manager should never be the sole custodian without escape hatches:

- **Recovery phrase:** Users receive a human-readable backup at account creation. This can reconstruct the keypair on any gateway or smartphone app.
- **On-chain recovery:** A time-locked recovery mechanism allows users to rotate their key if their Account Manager disappears, using their recovery phrase + a new gateway.
- **Keypair compatibility:** The same keypair format (sr25519) is used as in the smartphone app. A user who later acquires a smartphone can import their recovery phrase into the Encointer app and gain full self-custody.

### 3. TEE-Based Account Managers

Account Managers can run inside Trusted Execution Environments (Intel SGX, ARM TrustZone) so that even the gateway operator cannot extract private keys. This is consistent with Encointer's existing use of TEEs for sidechain workers.

```
┌─────────────────────────────────────┐
│         Account Manager Host        │
│  ┌───────────────────────────────┐  │
│  │     TEE (SGX Enclave)        │  │
│  │                               │  │
│  │  - Private key storage       │  │
│  │  - Transaction signing       │  │
│  │  - ZK proof generation       │  │
│  │  - PIN verification          │  │
│  │                               │  │
│  │  Keys never leave enclave    │  │
│  └───────────────────────────────┘  │
│                                     │
│  - USSD session management          │
│  - SMS notifications                │
│  - Blockchain RPC calls             │
└─────────────────────────────────────┘
```

The gateway operator handles routing and session management but cannot access user keys or sign transactions without the user's PIN (which is verified inside the enclave).

### 4. Threshold Custody (Advanced)

For high-value accounts, a multi-gateway threshold scheme is possible:
- The user's private key is split across 2-of-3 Account Managers using Shamir's Secret Sharing
- Any two gateways must cooperate to sign a transaction
- No single gateway can unilaterally move funds
- Adds latency but dramatically reduces trust assumptions

### 5. Community-Operated Infrastructure

Encointer communities can operate their own gateways as part of their governance:
- The community votes on who operates the gateway
- Gateway operators are accountable to the community
- Revenue from USSD fees (if any) flows back to the community treasury
- This mirrors how M-Pesa agents operate as community-embedded infrastructure

## Security Considerations

### SIM Swap Attacks

Phone number alone must never be sufficient for account access. Mitigations:
- **Mandatory PIN:** Every sensitive operation requires PIN entry
- **Rate limiting:** Failed PIN attempts trigger exponential backoff and eventual lockout
- **SMS alerts:** Any login from a new SIM triggers an SMS to the last known number
- **Cool-down periods:** After a SIM change, large transfers are delayed 24-48 hours
- **Recovery phrase:** Ultimate recovery bypasses phone number entirely

### Account Manager Compromise

If a gateway is compromised:
- **TEE protection:** Keys in enclaves are not extractable even with root access
- **Gateway diversity:** Users on other gateways are unaffected
- **Detection:** Unusual transaction patterns trigger community alerts
- **Recovery:** Users can migrate to another gateway using their recovery phrase
- **Auditability:** All gateway-signed transactions are on-chain and publicly auditable

### Ceremony Fraud

Feature phone participants have weaker attestation than smartphone users:
- **Minimum smartphone ratio:** Require at least 1 smartphone user per meetup to anchor verification
- **Reputation weighting:** Feature phone attestations could carry lower initial reputation weight until the user builds history
- **Progressive trust:** After N successful ceremonies, a feature phone user's attestation weight increases
- **Cell-ID verification:** Cross-reference reported meetup location with cell tower data

### USSD Session Hijacking

USSD sessions travel unencrypted over the GSM air interface:
- **PIN for every transaction:** Even if a session is hijacked, the attacker needs the PIN
- **Session tokens:** Short-lived, single-use session tokens prevent replay
- **No sensitive data in USSD display:** Balances are shown, but private keys and proofs never traverse the USSD channel

## What Changes On-Chain vs. Off-Chain

### Unchanged (On-Chain)

- All pallets: ceremonies, balances, communities, bazaar, democracy, faucet, offline-payment
- Ceremony cycle and phases (registering, assigning, attesting)
- Demurrage calculation
- Reputation system
- Groth16 proof verification for offline payments
- Community currency issuance rules

### New (Off-Chain Only)

- USSD Gateway service (session management, menu rendering)
- Account Manager service (key custody, transaction signing, proof generation)
- Phone-number-to-account mapping registry
- SMS notification service (transaction receipts, ceremony reminders)
- Short attestation code generation for hybrid meetups
- Offline payment lookup code service

### Minor On-Chain Additions (Optional)

- **Gateway registry pallet:** Communities can register approved gateways and their TEE attestations on-chain, making trust assumptions transparent
- **Attestation type flag:** Mark whether an attestation came from a smartphone (full GPS+camera) or feature phone (code+co-attestation), allowing communities to set policy on minimum smartphone ratios
- **Phone hash mapping:** An optional on-chain mapping from `hash(phone_number)` to account ID, enabling USSD users to send to phone numbers without the gateway being the sole directory. Privacy-preserving: only the hash is stored.

## Implementation Phases

### Phase 1: Transfers Only
- Account creation via USSD
- Balance queries
- Send/receive community currency
- SMS receipts
- **Minimum viable product** — already useful for daily commerce

### Phase 2: Ceremony Participation
- Ceremony registration via USSD
- Hybrid meetups with smartphone co-attestation
- UBI claiming
- Reputation building
- **Unlocks the core value proposition** — UBI for feature phone users

### Phase 3: Full Feature Set
- Offline payments via gateway-assisted ZK proofs
- Bazaar listings and purchases
- Community governance voting
- Faucet claims
- Multi-gateway portability

### Phase 4: Decentralization Hardening
- TEE-based Account Managers
- Threshold custody option
- On-chain gateway registry
- Community-operated infrastructure
- Cell-ID location verification partnerships

## Comparison: Smartphone vs. USSD

| Feature | Smartphone App | USSD |
|---|---|---|
| Self-custody | Yes (local keys) | No (gateway-custodied) |
| Ceremony participation | Full (GPS, camera, BLE) | Hybrid (code-based, needs smartphone co-attestor) |
| Transfers | QR code or address | Phone number or short ID |
| Offline payments | Local ZK proof generation | Gateway-generated proofs |
| Bazaar | Rich browsing | Text-only, abbreviated |
| Governance | Full proposal view | Summary + SMS for details |
| Location proof | GPS | Cell-ID + co-attestation |
| Internet required | Yes | No (GSM only) |
| Works on | Smartphones | Any GSM phone |

## Open Questions

1. **Telco partnerships:** USSD short codes require agreements with mobile network operators. How to negotiate these across many countries? Could a shared short code model (like `*384*` prefix) work across operators?

2. **Gateway economics:** Who pays for running Account Managers? Options: community treasuries, small per-transaction fees (in community currency), NGO grants, telco partnerships.

3. **Purely feature-phone meetups:** Can a ceremony meetup work with zero smartphones? The voice-call + DTMF + cell-ID approach is weaker. Should there be a hard minimum smartphone requirement?

4. **Regulatory compliance:** In many jurisdictions, USSD financial services require money transmitter or e-money licenses. How does this interact with Encointer's decentralized nature?

5. **PIN recovery:** If a user forgets their PIN but has their recovery phrase, the recovery phrase alone should suffice to restore access. But how to make recovery phrases accessible to users who may not be literate? Voice-based recovery?

6. **Offline payment code size:** Can the lookup-code approach be made trustless? E.g., store the proof on IPFS with the CID as the lookup code, so any node can retrieve it.

## Conclusion

Encointer can be made accessible to feature phone users via USSD without changing the on-chain protocol. The key trade-off is **self-custody vs. accessibility**: USSD users must trust an Account Manager with their keys, but this trust can be mitigated through TEEs, multiple independent gateways, account portability, and community governance.

The design preserves Encointer's core properties:
- **Proof-of-personhood** still requires physical presence at meetups
- **Community currencies** and **demurrage** work identically
- **Democratic governance** remains one-person-one-vote
- **Decentralization** is maintained through gateway diversity and on-chain verifiability

The approach follows M-Pesa's proven model: meet users where they are (basic phones, no internet), and let the infrastructure handle the complexity. The crucial difference is that Encointer's on-chain transparency ensures that even custodial gateways cannot forge personhood or inflate currency supply — the blockchain remains the single source of truth.
