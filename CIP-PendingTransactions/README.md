---
CIP: ?
Title: Pending Transactions - Two-Phase Transaction Protocol (2PT)
Status: Proposed
Category: Wallets
Authors:
  - "[Yurelle Gamier] <yurelle.gamier@gmail.com>"
Implementors: []
Discussions:
  - https://forum.cardano.org/t/cip-proposal-pending-transactions-a-two-phase-transaction-protocol-for-safer-crypto-sends/153742
Created: 2026-03-23
License: CC-BY-4.0
---

## Abstract

This document specifies a protocol for pending cryptocurrency transactions, wherein a sender may reserve funds for a recipient without immediately and irrevocably transferring them. The sender may subsequently either confirm the transfer, making it permanent, or abort it, returning the funds to their own wallet. The recipient is notified of and can observe the reserved funds throughout. Authority to confirm, abort, or decline may be delegated to designated parties, enabling trustless escrow, automated merchant verification, and bilateral negotiation flows. This protocol is designed to be implementable on any cryptocurrency network capable of expressing fund locking and optional time-bounded state transitions, and is motivated by significant usability, safety, and commercial deficiencies in current one-phase, irreversible transaction models.

This specification is organized into three compliance tiers. Tiers are cumulative: each tier requires full implementation of all prior tiers. Tier 1 defines the minimum viable protocol suitable for conservative chains. Tier 2 extends with authority delegation, counterparty decline, and capability tokens. Tier 3 adds amendment negotiation and advanced escrow patterns.

---

## Motivation: why is this CIP necessary?

### The Problem with Irreversible Transactions

Current cryptocurrency transaction models are, by design, one-shot and irreversible. Once a sender broadcasts a transaction and it is confirmed on-chain, the funds are gone. There is no undo, no "did you mean this address?", no grace period. This is often cited as a feature (ex: censorship resistance, finality, trustlessness) and in many respects it is. However, it creates a class of severe, unrecoverable user errors that have no analog in traditional finance, and which impose a meaningful and largely unnecessary psychological and practical burden on users, particularly laymen.

The consequences are not theoretical. Funds sent to a mistyped address, a stale address, or a wrong network are permanently and irrecoverably lost. No support line exists. No chargeback is possible. The current model asks every user (from the most sophisticated developer to a first-time participant) to be perfectly correct, every single time, with no safety net whatsoever.

This specification proposes that cryptocurrency networks adopt a **two-phase transaction protocol** (AKA "Pending Transactions") that preserves all the trustless and decentralized properties of existing transactions, while introducing a reversible preparation phase that dramatically reduces the risk and psychological burden of sending funds.

---

### Use Case 1: Lowering the Barrier to Entry for Lay Users

The all-or-nothing nature of cryptocurrency transactions is one of the most significant psychological barriers preventing mainstream adoption. A user who has never sent cryptocurrency before faces the following situation: they copy a long, opaque string of characters, paste it into a send field, enter an amount, and press a button; after which their money is either where they intended, or gone forever, with no intermediate feedback, no confirmation from the recipient, and no recourse.

This is an unreasonable ask. No other payment system works this way. Traditional banking offers confirmation screens, email notifications to both parties, and reversal windows. Even cash transactions have the physical reality of handing money to a person in front of you and watching them receive it.

Users of traditional banking are already familiar with the concept of a pending transaction; a charge that has been authorized but not yet fully settled. This specification brings that familiar and trusted mechanic to cryptocurrency. The Two-Phase Transaction Protocol allows a lay user to initiate a pending send (reserving their funds without releasing them) and only committing once they have received explicit confirmation that the destination is correct. The funds are visibly reserved, the recipient can see them incoming, and the sender retains the ability to cancel if anything looks wrong. This transforms "click once and either it worked or you're ruined" into a recognizable two-step process: *send intent → receive confirmation → commit*.

---

### Use Case 2: Bilateral Address Verification

A primary source of irreversible loss in cryptocurrency is address error (ex: mistyped characters, copied wrong addresses, addresses for the wrong network, or addresses belonging to wallets the recipient no longer controls). Currently, the only protection available is manual, error-prone visual inspection of a long alphanumeric string by the sender alone, prior to sending.

The two-phase protocol enables a fundamentally better verification model: the recipient can confirm the pending transaction before the sender commits it. Because the pending transaction is observable on-chain, the recipient can verify that the reserved amount is correct, that it is destined for an address they control, and that it is on the correct network; and communicate this verification back to the sender before any funds are irreversibly moved. The sender, having received this confirmation, can then commit with confidence. If the recipient does not recognize the transaction, or sees an error, the sender aborts and no funds are lost.

This transforms address verification from a unilateral, error-prone manual step into a bilateral, cryptographically grounded handshake.

---

### Use Case 3: Cryptographic Proof of Funds with Guaranteed Persistence

In commercial and contractual contexts (ex: real estate, large purchases, service agreements, escrow arrangements) a party is often required to demonstrate that they have sufficient funds available and that those funds are genuinely committed to a transaction. Current cryptocurrency offers no standardized mechanism for this. A sender can show a wallet balance, but that balance can be spent at any moment after the snapshot is taken. There is no way to prove that funds shown are still reserved for the stated purpose.

A pending two-phase transaction provides exactly this guarantee. The locked funds are verifiable on-chain, cannot be double-spent while the prepare phase is active, and are associated with a specific, immutable transaction ID that the recipient can monitor in real time. The recipient knows:

- The funds exist
- They are locked and cannot be moved to a third party
- They are explicitly designated for this transaction
- They will either be confirmed (completing the payment) or aborted/declined (in which case the recipient is immediately notified by the state change on-chain)

This is a stronger proof-of-funds than any existing cryptocurrency mechanism, and competitive with (or superior to) traditional escrow in terms of trustlessness and transparency. For long-term commitments such as real estate or multi-stage contracts, a pending transaction with no timeout provides a persistent, monitorable financial commitment for the full duration of the process.

---

### Use Case 4: Automated Commercial Payment Systems

Businesses that accept cryptocurrency payments (ex: e-commerce platforms, subscription services, point-of-sale systems) currently face significant friction in building reliable payment flows. They must trust that an incoming payment is correct, poll for confirmations, and handle edge cases where payments arrive malformed or incomplete, with no ability to communicate back to the sender before funds are irrevocably committed.

The two-phase protocol enables a far richer automated payment flow:

1. The customer initiates a PREPARE transaction designating the merchant's address and the expected amount, including a capability token granting the merchant commit authority; if the merchant DECLINE path (step 8) is desired, the customer may also grant the merchant `decline_authority` at this stage
2. The merchant's automated system detects the pending transaction on-chain
3. The system verifies: correct destination address, correct amount, correct network
4. If verification passes, the system uses its delegated commit authority to confirm the transaction automatically; simultaneously, it may reserve inventory, generate an order confirmation, send the customer an email confirmation, initiate fulfillment preparation, or trigger any other downstream business logic
5. The customer need take no further action in the happy path. The experience is comparable to a debit card transaction
6. If verification fails, the system notifies the customer of the specific error (wrong amount, wrong address, etc.) and instructs them to abort; no funds are lost; downstream reservations are not made; a notification email may be sent describing the failure
7. If the customer aborts or the transaction times out, the system automatically cancels any reservations, revokes access to digital goods, cancels pending shipments, and may send the customer a notification email confirming the cancellation
8. If the merchant's system detects a problem after PREPARE that prevents fulfillment (ex: fraud detection, items going out of stock, or a compliance hold) and has been granted `decline_authority`, it may issue a DECLINE, returning funds immediately to the customer and triggering the same downstream cancellation logic; a notification email may be sent explaining the reason for refusal

This enables cryptocurrency payment systems to have the reliability and user experience comparable to traditional payment processors, without sacrificing decentralization or requiring trusted intermediaries.

---

## Specification

This specification defines the Two-Phase Transaction Protocol in three compliance tiers. Tiers are cumulative: each tier requires full implementation of all prior tiers.

---

### TIER 1 - Core Protocol

*Minimum viable implementation. All conforming 2PT implementations must support Tier 1.*

#### State Machine (Tier 1)

```
[NONE] ──── PREPARE() ────► [PENDING]
                                │
                    ┌───────────┴───────────┐
                    │                       │
               CONFIRM()          ABORT() / auto-abort on timeout
                    │                       │
                    ▼                       ▼
             [CONFIRMED]             [ABORTED]
```

CONFIRMED and ABORTED are terminal states. No further operations are valid after either.

---

#### Operation: PREPARE

**Initiator:** Sender

**Parameters:**
- `sender_address`       - the originating wallet address
- `recipient_address`    - the intended destination wallet address
- `amount`               - the quantity of currency to be reserved
- `timeout` *(optional)* - block height or timestamp after which auto-abort triggers; if omitted, the transaction remains pending indefinitely until it reaches a terminal state
- `memo`    *(optional)* - human-readable or machine-readable annotation

**Effect:**
- The specified amount is deducted from the sender's spendable balance
- The funds are locked in a protocol-defined pending state; they may not be spent, transferred, or used as input to any other transaction while in this state
- The pending transaction is observable on-chain and queryable by any party in possession of the transaction ID
- The recipient's wallet reflects the pending inbound amount as reserved-but-not-yet-spendable
- If `timeout` is specified, an auto-abort is scheduled as described below

**Produces:**
- `transaction_id` - a unique, immutable identifier for this pending transaction, usable for monitoring, reference, and capability token targeting

---

#### Operation: CONFIRM

**Initiator:** Sender *(Tier 1; authority model extended in Tier 2)*

**Precondition:** Transaction must be in PENDING state

**Effect:**
- Funds are irrevocably transferred to `recipient_address`
- Transaction state transitions to CONFIRMED
- Funds become spendable by recipient

---

#### Operation: ABORT

**Initiator:** Sender *(Tier 1; authority model extended in Tier 2)*

**Precondition:** Transaction must be in PENDING state

**Effect:**
- Locked funds are returned to `sender_address` and become immediately spendable
- Transaction state transitions to ABORTED

---

#### Timeout and Auto-Abort

If `timeout` is specified in PREPARE and the transaction remains in PENDING state past that timeout without reaching a terminal state, the protocol automatically transitions the transaction to ABORTED and returns funds to the sender. Auto-abort produces the ABORTED state identically to an explicit ABORT operation.

Timeout is optional. For long-term use cases such as real estate, escrow arrangements, or multi-stage contracts, an indefinite pending state is a legitimate and intended configuration. Wallet implementations are encouraged to surface a clear indication to both parties when no timeout is set, so that both are aware the transaction will not auto-resolve.

---

#### Observability Requirements (Tier 1)

A conforming Tier 1 implementation must guarantee that any party in possession of a `transaction_id` can query the following from the network without requiring trust in any third party:

- Current transaction state: PENDING / CONFIRMED / ABORTED
- `sender_address`
- `recipient_address`
- `amount`
- `timeout` (if set)

The `memo` field, if set, is not required to be publicly queryable by this specification, as it may contain sensitive payment information. Whether `memo` is observable on-chain is implementation-defined. Implementations that expose `memo` publicly should document this clearly so that users understand memo contents are not private.

---

#### Double-Spend Prevention

Funds locked by a PREPARE operation must not be available for any other transaction (including another PREPARE) until the current transaction reaches a terminal state. Implementations must enforce this at the protocol level.

---

### TIER 2 - Authority Delegation

*Extends Tier 1. Chains implementing Tier 2 must also fully implement Tier 1.*

#### Overview

Tier 2 introduces configurable authority over the CONFIRM, ABORT, and DECLINE operations, enabling delegation to recipients, third-party arbitrators, or requiring mutual consent. It also introduces the DECLINE operation (non-sender refusal, by the recipient or a designated authority) and capability tokens for cryptographically verified delegated authority.

---

#### Authority Model

Three authority parameters are added to PREPARE:

- `commit_authority`  - governs who may execute CONFIRM
- `abort_authority`   - governs who may execute ABORT
- `decline_authority` - governs who may execute DECLINE

`commit_authority` accepts all five values listed below. `abort_authority` and `decline_authority` each omit one value for semantic reasons described beneath the table.

|--------------------|---------------------------------------------------------------------|
| Value              | Meaning                                                             |
|--------------------|---------------------------------------------------------------------|
| `SENDER_ONLY`      | Only the sender may execute this operation                          |
| `RECIPIENT_ONLY`   | Only the recipient may execute this operation                       |
| `THIRD_PARTY_ONLY` | Only the designated third party may execute this operation          |
| `MUTUAL`           | All designated parties must co-sign to execute this operation       |
| `EITHER`           | Any single designated party may execute this operation unilaterally |
|--------------------|---------------------------------------------------------------------|

`abort_authority` accepts all values **with the exception of `RECIPIENT_ONLY`**. A recipient executing ABORT would produce an ABORTED terminal state, which observers and wallets associate with sender-initiated cancellation; a misrepresentation of who terminated the transaction. Implementations wishing to give the recipient unilateral termination authority should use `decline_authority = RECIPIENT_ONLY` instead, which produces the DECLINED state and correctly attributes the termination to the recipient. The valid values for `abort_authority` are therefore: `SENDER_ONLY`, `THIRD_PARTY_ONLY`, `MUTUAL`, and `EITHER`.

`decline_authority` accepts all values **with the exception of `SENDER_ONLY`**, which is omitted because a sender executing DECLINE would be semantically identical to the sender executing ABORT; both return funds to the sender and terminate the transaction. That behavior is already provided by ABORT in Tier 1. Allowing `decline_authority = SENDER_ONLY` would produce a DECLINED state in response to a sender-initiated action, misrepresenting to observers that the recipient refused the transaction. The valid values for `decline_authority` are therefore: `RECIPIENT_ONLY`, `THIRD_PARTY_ONLY`, `MUTUAL`, and `EITHER`.

**Note on `decline_authority = EITHER`:** Even when set to `EITHER`, the sender is never an eligible executing party for DECLINE, for the same semantic reasons that `SENDER_ONLY` is excluded. For `decline_authority`, `EITHER` means any single party among the recipient and the designated third party (if set) may execute DECLINE unilaterally.

**Default values if omitted:**
- `commit_authority  = SENDER_ONLY`
- `abort_authority   = SENDER_ONLY`
- `decline_authority = RECIPIENT_ONLY`

These defaults preserve full Tier 1 behavior when Tier 2 parameters are not used.

An optional `third_party_address` parameter may be specified in PREPARE to designate an arbitrator. This parameter is required if any authority value references `THIRD_PARTY`.

**Designated parties for MUTUAL:** When any authority parameter is set to `MUTUAL`, the co-signing requirement applies to: the sender, the recipient, and the third party if `third_party_address` is specified. For `decline_authority = MUTUAL`, the sender is excluded from the co-signing requirement for the same semantic reasons described above; only the recipient and the designated third party (if set) must co-sign. A MUTUAL operation is not valid until all applicable designated parties have co-signed. The protocol mechanism for accumulating co-signatures toward a MUTUAL operation is chain-specific and is not defined in this specification; see Open Questions.

**Authority configuration is immutable once PREPARE is on-chain.** Authority parameters cannot be modified by any subsequent operation, including amendment proposals (see Tier 3). Any change to authority configuration requires the current transaction to be ABORTED and a new PREPARE issued with the desired configuration. This restriction exists because authority parameters define the fundamental trust topology of the transaction; silent or unilateral changes to them would create severe security vulnerabilities.

---

#### Warnings for Dangerous Configurations

The following configurations materially alter the balance of power between parties and carry elevated risk. Wallet implementations must surface prominent warnings when any of these are selected:

**`abort_authority = THIRD_PARTY_ONLY`:** The sender loses the unilateral ability to cancel the transaction. Appropriate for escrow arrangements where the sender's abort privilege would undermine the recipient's trust, but should be used only with thoroughly vetted arbitrators.

**`commit_authority = THIRD_PARTY_ONLY`:** Neither principal can unilaterally finalize the transaction. Appropriate when an independent condition must be verified before funds are released.

**`abort_authority = EITHER` or `commit_authority = EITHER`:** Any single designated party may execute the operation unilaterally. For abort, this means a malicious or negligent third party could sabotage a legitimate transaction without the principals' consent. For commit, this means a malicious or negligent third party could finalize a transaction prematurely. Use only with parties whose incentives are fully aligned with the transaction.

---

#### Observability Requirements (Tier 2 Extension)

In addition to the Tier 1 observability requirements, a conforming Tier 2 implementation must guarantee that the following are queryable by any party in possession of the `transaction_id`:

- Current transaction state: PENDING / CONFIRMED / ABORTED / **DECLINED** *(new in Tier 2)*
- `commit_authority`
- `abort_authority`
- `decline_authority`
- `third_party_address` (if set)
- `commit_auth_pubkey` (if set)
- `abort_auth_pubkey` (if set)
- `decline_auth_pubkey` (if set)

This ensures that recipients can verify their own authority configuration, that automated systems can confirm their capability token was correctly embedded, and that third-party observers can validate the full trust topology of the commitment for proof-of-funds purposes.

---

#### Capability Tokens (Delegated Operation Authorization)

To enable automated or third-party execution of CONFIRM, ABORT, or DECLINE without relying solely on address-based identity verification, a capability token mechanism is defined. Capability tokens provide cryptographic proof that the executing party is the one authorized at PREPARE time, preventing unauthorized parties from impersonating a designated authority.

**Parameters added to PREPARE:**
- `commit_auth_pubkey`  *(optional)* - public key authorizing CONFIRM; required if `commit_authority` delegates to any non-sender party
- `abort_auth_pubkey`   *(optional)* - public key authorizing ABORT;   required if `abort_authority` delegates to any non-sender party
- `decline_auth_pubkey` *(optional)* - public key authorizing DECLINE; required if `decline_authority` delegates to any non-recipient party

**Flow (illustrated for commit; abort and decline follow the same pattern):**

1. Before PREPARE is issued, the intended commit authority generates a **one-time keypair** specific to this transaction
2. The public key is shared with the sender via any out-of-band channel (payment request URL, QR code, API response, etc.)
3. The sender includes this public key as `commit_auth_pubkey` in the PREPARE parameters
4. Upon detecting the pending transaction on-chain, the commit authority verifies the transaction parameters (correct address, correct amount, correct network, etc.)
5. If verification passes, the commit authority signs a CONFIRM operation using the corresponding private key; the protocol validates this signature against `commit_auth_pubkey` and executes CONFIRM
6. If verification fails, the commit authority may instruct the sender to ABORT, or may execute DECLINE if `decline_authority` permits

A party that receives a pending transaction accidentally (i.e. one that was never the intended recipient and never participated in the keypair exchange) cannot execute CONFIRM because they do not hold the private key paired with `commit_auth_pubkey`. This prevents malicious addresses from auto-committing accidental or misdirected pending transactions.

Capability tokens are one-time use. A new keypair must be generated for each transaction. Importantly, if a transaction is replaced via ACCEPT_AMEND (see Tier 3), the new transaction inherits all original authority pubkeys (`commit_auth_pubkey`, `abort_auth_pubkey`, `decline_auth_pubkey`); the corresponding private keys remain valid against the new transaction. However, the new transaction has a new `transaction_id` and potentially amended parameters. Any automated system holding commit, abort, or decline authority must re-verify the new transaction's parameters before signing any operation against it, and must not act based on approval or decisions made against the prior (now REPLACED) transaction.

---

#### Operation: DECLINE

**Initiator:** Party authorized per `decline_authority` *(default: `RECIPIENT_ONLY`)*

**Precondition:** Transaction must be in PENDING state

**Effect:**
- Locked funds are returned to `sender_address` and become immediately spendable
- Transaction state transitions to DECLINED

DECLINED is a terminal state functionally identical to ABORTED in terms of fund return, but distinguished in the state record and wallet UX to indicate that a non-sender party terminated the transaction. By design, DECLINE can never be sender-initiated (see Authority Model); the DECLINED state therefore reliably signals to observers that the transaction was refused by the recipient or a designated authority, not cancelled by the sender. Wallet implementations should surface this distinction clearly, as ABORTED and DECLINED carry different implications (sender self-cancellation vs. active refusal by the counterparty).

---

#### Updated State Machine (Tier 2)

```
[NONE] ──── PREPARE() ────► [PENDING]
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
       CONFIRM()            ABORT() /            DECLINE()
   (per commit_auth)      auto-abort on      (per decline_auth;
           │                 timeout         sender never eligible)
           │                    │                    │
           ▼                    ▼                    ▼
      [CONFIRMED]           [ABORTED]            [DECLINED]
```

---

### TIER 3 - Amendment and Negotiation

*Extends Tier 2. Chains implementing Tier 3 must also fully implement Tiers 1 and 2.*

#### Overview

Tier 3 introduces the ability for parties to propose amendments to a pending transaction's parameters - enabling negotiation, adjustment, and renegotiation without aborting and reissuing transactions from scratch. It is designed for long-running commitments where terms may evolve before finalization.

#### Principals

Throughout Tier 3, the term **principal** refers to any party that is a named participant in a given transaction: the sender, the recipient, and the designated third party if `third_party_address` was set in the original PREPARE. An arbitrary address that does not match one of these three roles is not a principal of the transaction and may not initiate or respond to any Tier 3 operation against it.

---

#### Operation: PROPOSE_AMEND

**Initiator:** Any principal of the transaction

**Precondition:** Transaction must be in PENDING state. The initiator must be a principal: the sender, the recipient, or the designated third party (whose address must match `third_party_address` from the original PREPARE).

**Parameters:**
- `target_transaction_id`         - the transaction being amended
- `proposed_amount`  *(optional)* - new amount
- `proposed_timeout` *(optional)* - new timeout value for the transaction; may freely shorten, extend, or remove the existing timeout; not clamped
- `proposed_memo`    *(optional)* - new memo; to explicitly clear an existing memo, set this to an empty value; omitting this parameter entirely leaves the existing memo unchanged
- `proposal_timeout` *(optional)* - block height or timestamp after which this proposal auto-lapses if not accepted, rejected, or retracted; clamped to the parent transaction's current timeout if one exists

**Effect:**
- A proposal record is created and associated with `target_transaction_id`
- The target transaction's state is **not modified** - it remains PENDING
- Multiple proposals may be simultaneously in-flight against the same transaction, from any combination of principals
- The proposal record stores the `proposer_address` of the initiating principal, queryable by any party in possession of the `proposal_id` or the parent `transaction_id`; this allows principals receiving multiple simultaneous proposals to identify their source
- The proposal enters PROPOSAL_PENDING state

**Produces:**
- `proposal_id` - a unique, immutable identifier for this proposal, usable for querying proposal state and for targeting ACCEPT_AMEND, REJECT_AMEND, and RETRACT_AMEND operations

**Constraint - authority parameters are not amendable:** No party may propose an amendment that modifies `commit_authority`, `abort_authority`, `decline_authority`, `third_party_address`, `commit_auth_pubkey`, `abort_auth_pubkey`, or `decline_auth_pubkey`. Authority configuration is fixed at PREPARE and cannot be changed via amendment. Any change to authority configuration requires the current transaction to be terminated (via ABORT or DECLINE) and a new PREPARE issued. This restriction prevents malicious amendments from quietly stripping or reassigning trust configuration.

**Timeout clamping rules:**
- `proposed_timeout` is not clamped; an amendment may freely shorten, extend, or remove the parent transaction's timeout - this is the purpose of the parameter
- `proposal_timeout`, if specified and the parent transaction has a timeout, must not exceed the parent transaction's current timeout; if it does, it is silently clamped to that value; a proposal that outlives its parent transaction is meaningless since the parent's own auto-abort would terminate the proposal regardless
- If the parent transaction has no timeout, `proposal_timeout` is unclamped and may be set to any value
- The parent transaction's timeout, if set, is not paused during amendment negotiation; it continues independently of any in-flight proposals

---

#### Operation: ACCEPT_AMEND

**Initiator:** Any principal of the transaction *other than the proposer*

**Precondition:** Proposal must be in PROPOSAL_PENDING state; parent transaction must be in PENDING state

**Parameters:**
- `proposal_id` - the identifier of the proposal being accepted

**Effect:**
- The original transaction transitions directly to REPLACED (a terminal state); it does not pass through or enter the ABORTED state, even momentarily
- A new PREPARE is issued with the parameters specified in the proposal; any parameters not included in the proposal are inherited unchanged from the original transaction; all authority configuration is inherited unchanged from the original transaction
- The new PREPARE produces a new `transaction_id`
- All other in-flight proposals targeting the original `transaction_id` are simultaneously terminated (PROPOSAL_TERMINATED), as their target is now in a terminal state

**Rule:** No party may accept their own proposal. This applies without exception regardless of authority configuration.

**Race conditions:** Two race conditions are possible and both resolve via the same mechanism: the precondition that the parent transaction must be in PENDING state:

1. Two parties simultaneously accept the same proposal: the first acceptance confirmed on-chain executes; the second is rejected because the parent transaction is already in REPLACED state
2. Two parties simultaneously accept different proposals targeting the same parent transaction: the first acceptance confirmed on-chain executes; the second is rejected because the parent transaction is already in REPLACED state

In both cases no special handling is required beyond enforcing the precondition.

---

#### Operation: REJECT_AMEND

**Initiator:** Any principal of the transaction other than the proposer

**Precondition:** Proposal must be in PROPOSAL_PENDING state

**Parameters:**
- `proposal_id` - the identifier of the proposal being rejected

**Effect:**
- The proposal transitions to PROPOSAL_REJECTED (terminal)
- The parent transaction is unaffected and remains PENDING
- Other in-flight proposals against the same transaction are unaffected

---

#### Operation: RETRACT_AMEND

**Initiator:** The proposer only

**Precondition:** Proposal must be in PROPOSAL_PENDING state

**Parameters:**
- `proposal_id` - the identifier of the proposal being retracted

**Effect:**
- The proposal transitions to PROPOSAL_RETRACTED (terminal)
- The parent transaction is unaffected and remains PENDING
- Other in-flight proposals against the same transaction are unaffected

RETRACT_AMEND is the symmetric counterpart to REJECT_AMEND, allowing a proposer to withdraw their own proposal - for example, if they entered incorrect parameters or if off-chain circumstances have changed. Without this operation, a proposer with no `proposal_timeout` set would have no recourse to clear their own proposal until another principal rejects it or the parent transaction terminates.

---

#### Observability Requirements (Tier 3 Extension)

In addition to the Tier 1 and Tier 2 observability requirements, a conforming Tier 3 implementation must guarantee that the following are queryable by any party in possession of a `transaction_id` or `proposal_id`:

- Current transaction state: PENDING / CONFIRMED / ABORTED / DECLINED / **REPLACED** *(new in Tier 3)*
- Current proposal state: PROPOSAL_PENDING / PROPOSAL_ACCEPTED / PROPOSAL_REJECTED / PROPOSAL_RETRACTED / PROPOSAL_LAPSED / PROPOSAL_TERMINATED
- All proposal parameters (`proposed_amount`, `proposed_timeout`, `proposed_memo`, `proposal_timeout`) for any proposal targeting a given `transaction_id`
- `proposer_address` for any proposal, identifying which principal filed it
- The `transaction_id` of the replacement transaction produced by ACCEPT_AMEND, queryable from the REPLACED transaction record

---

#### Proposal Lifecycle and Timeouts

```
                           [PROPOSAL_PENDING]
                                   │
         ┌─────────────────────────┼─────────────────────────────┐
         │                         │                             │
   ACCEPT_AMEND              RETRACT_AMEND                REJECT_AMEND /
   (non-proposer            (proposer only)               proposal_timeout expires /
    principal)                     │                      parent transaction reaches
        │                          │                      any terminal state
        │                          │                             │
        ▼                          ▼                             ▼
[PROPOSAL_ACCEPTED]       [PROPOSAL_RETRACTED]           [PROPOSAL_REJECTED /
        │                                                  PROPOSAL_LAPSED /
        ▼                                                  PROPOSAL_TERMINATED]
(parent → REPLACED,
 new tx created,
 all sibling proposals
 → PROPOSAL_TERMINATED)
```

- If a proposal's `proposal_timeout` expires before it is accepted, rejected, or retracted, it lapses automatically (PROPOSAL_LAPSED) with no effect on the parent transaction
- If the parent transaction reaches any terminal state (CONFIRMED, ABORTED, DECLINED, REPLACED), all in-flight proposals targeting it are simultaneously terminated (PROPOSAL_TERMINATED) regardless of their individual timeouts
- The parent transaction's timeout, if set, is not paused during amendment negotiation; if the parent auto-aborts before a proposal is accepted, the proposal is terminated as a consequence

---

#### Updated State Machine (Tier 3)

```
Parent Transaction:
  [PENDING] ─────────────────────────────────────────────────────────────────────────►
      │                   │                   │                       │
  CONFIRM()            ABORT() /           DECLINE()             ACCEPT_AMEND()
(per commit_auth)     auto-abort on      (per decline_          (by non-proposer
      │                timeout           auth; sender            principal only)
      ▼                   │              never eligible)              │
[CONFIRMED]               ▼                   │                       ▼
                      [ABORTED]               ▼               [REPLACED] ──► new tx
                                          [DECLINED]         (all sibling proposals
                                                              → PROPOSAL_TERMINATED)

                                                              
                                                              
Amendment Proposals (independent, non-blocking, multiple in-flight allowed):
[PROPOSAL_PENDING] ──► ACCEPT_AMEND (non-proposer principal)  ──► [PROPOSAL_ACCEPTED]
                   ──► REJECT_AMEND (non-proposer principal)  ──► [PROPOSAL_REJECTED]
                   ──► RETRACT_AMEND (proposer only)          ──► [PROPOSAL_RETRACTED]
                   ──► proposal_timeout expires               ──► [PROPOSAL_LAPSED]
                   ──► parent reaches terminal state          ──► [PROPOSAL_TERMINATED]
```

---

### Versioning

This specification uses semantic versioning. The current version is **1.0**.

Backward-compatible additions (new optional parameters, new optional operations, clarifications) constitute minor version increments. Changes that alter the semantics of existing operations, remove operations, or require re-implementation of existing behavior constitute major version increments and must be addressed in a superseding CIP. Chains implementing a given tier of this specification at version 1.x remain compliant with any 1.y where y > x.

---

## Rationale: how does this CIP achieve its goals?

### Design Decisions

**Two-phase over single-phase:** The existing one-shot irreversible model is retained as the default for all existing tooling. 2PT is strictly additive; a Tier 1 implementation with default authority settings is a strict superset of the current model, requiring no changes to existing wallets or workflows that choose not to adopt it.

**Tiered compliance:** The three-tier structure reflects the reality that chains have different capacity and appetite for protocol changes. Tier 1 is deliberately narrow enough for even conservative chains like Bitcoin to consider, while Tiers 2 and 3 provide the richer feature set that more expressive chains like Cardano can implement in full. This avoids an all-or-nothing proposal that would be rejected by conservative chains and ignored by expressive ones.

**Counterparty decline vs. sender abort:** These are distinct terminal states rather than a single "cancelled" state because they carry different semantic meaning for all observers; a seller declining is commercially and legally different from a buyer cancelling. Collapsing them would destroy information that downstream systems (merchant platforms, escrow services, dispute resolution) depend on.

**Authority immutability:** Authority configuration is fixed at PREPARE and cannot be amended. This is a deliberate security decision: allowing amendments to alter trust topology would create an attack vector where a malicious amendment quietly reassigns who controls confirmation or cancellation. The cost (requiring an abort and re-prepare to change authority) is low; the security benefit is high.

**Capability tokens over address-only identity:** Address-based identity verification is insufficient for automated systems because any party who discovers a pending transaction targeting their address could attempt to act on it. One-time keypairs ensure only the party who participated in the pre-transaction handshake can execute delegated operations, closing the malicious-address auto-commit attack vector.

**Amendment proposals as non-blocking:** In-flight proposals do not modify the parent transaction's state. This prevents a griefing attack where a party blocks a transaction indefinitely by filing repeated amendment proposals. The parent transaction continues toward its own timeout and can be acted upon at any time regardless of pending proposals.

### Relationship to Existing Mechanisms and Considered Alternatives

The following mechanisms occupy adjacent or overlapping design space. Each was considered as a basis or alternative for this proposal.

**HTLCs (Hash Time-Locked Contracts):** HTLCs, used extensively in the Lightning Network, provide time-bounded conditional payments contingent on revelation of a cryptographic preimage. They share timeout and reversibility properties with this proposal but are designed for trustless multi-hop routing; the lock condition is a hash preimage enabling atomic cross-party settlement, not a mechanism for human address verification or bilateral handshaking. HTLCs do not expose a recipient-visible pending state in standard wallet UX and do not address the use cases described in this document. HTLCs and 2PT are complementary rather than competitive.

**BIP-345 / OP_VAULT:** OP_VAULT proposes a mechanism for unilateral reversible Bitcoin transactions motivated by theft recovery. A vault owner may initiate an unvaulting which starts a time-delayed window during which a recovery address can claw funds back. This shares the reversible-window concept with Tier 1 of this proposal but is strictly unilateral; the recipient plays no role, and there is no bilateral handshake, recipient-visible pending state, or delegated authority model. OP_VAULT solves a different problem and does not resolve address verification, proof of funds, automated merchant flows, or negotiation. This proposal may be implemented as a complement to OP_VAULT on chains where OP_VAULT is available.

**Payment Channels:** Payment channels allow repeated off-chain transactions between two parties with periodic on-chain settlement, optimized for high-frequency bilateral payments. They require upfront channel establishment and are not designed for the general-purpose single-transaction safety and verification use cases described here. Payment channels and 2PT are not in conflict and may be used together.

**Existing Smart Contract Escrow (EVM Chains):** Bespoke escrow smart contracts on EVM-compatible chains have implemented subsets of the patterns described in this specification for years, demonstrating that the core mechanics are technically sound. The gap is not feasibility but standardization; no common interface, wallet UX convention, or cross-chain specification exists. This proposal provides that standard layer.

**Three-phase model (mandatory recipient acknowledgment):** A design requiring an explicit on-chain recipient ACCEPT before funds can be confirmed was considered and rejected for the base protocol. It would impose an additional on-chain operation cost on every transaction and create a new failure mode where the recipient goes dark after the sender commits. The 3-phase behavior is available as an opt-in configuration (`commit_authority = MUTUAL`) for parties who require mutual consent, without making it the default.

### Backward Compatibility

This proposal introduces no changes to existing transaction semantics. Standard single-phase transactions are unaffected. Wallets and nodes that do not implement 2PT continue to function normally. 2PT transactions appear as locked UTXOs (on UTXO chains) or escrow contract interactions (on account-based chains) to non-implementing systems, which may display them as "unspendable" or "in contract" depending on the chain's existing tooling; a well-understood and non-disruptive presentation.

---

## Implementation Notes (Non-Normative)

**UTXO-based chains (Bitcoin):** PREPARE locks one or more UTXOs to a script accepting CONFIRM, ABORT, or DECLINE redeemers, each gated by signature validation per the configured authority model and corresponding pubkeys. Timeout enforcement via `OP_CHECKLOCKTIMEVERIFY`. Tier 1 may be achievable via PSBT-based wallet coordination without a protocol change; Tier 2 capability tokens likely require covenant opcodes. BIP-345 (OP_VAULT) provides relevant prior art and potentially useful primitives for Tier 1.

**EUTXO-based chains (Cardano):** Cardano's EUTXO model and native Plutus validator scripts are well-suited to express all three tiers of this protocol. A PREPARE maps to a script-locked UTXO; CONFIRM, ABORT, and DECLINE map to redeemers gated by signature validation per the authority configuration and capability token pubkeys. Timeout enforcement via native slot validity ranges. Tier 1 and Tier 2 are likely implementable as wallet-level standards without protocol changes, making Cardano the natural reference implementation chain. Tier 3 amendment proposals can be expressed as additional UTXOs referencing the parent transaction ID.

**Account-based chains (Ethereum/EVM):** PREPARE calls a standard escrow contract holding funds and exposing CONFIRM, ABORT, and DECLINE functions gated by the authority configuration and pubkey verification. All three tiers are implementable as Solidity contracts today. Standardization of the contract interface (ABI) and wallet UX conventions is the primary remaining work.

---

## Path to Active

### Acceptance Criteria

This CIP becomes Active when:

1. At least one Cardano wallet (Eternl, Lace, Typhon, Yoroi, or equivalent) ships a production implementation of Tier 1 compliance, including on-chain observability of pending transaction state
2. A reference Plutus validator script implementing Tier 1 is reviewed and ratified by the Cardano developer community
3. No unresolved security objections remain open in the CIP discussion thread

Tier 2 and Tier 3 compliance are not required for Active status. They may be addressed in subsequent CIPs or in a revision to this CIP once Tier 1 is established.

### Implementation Plan

1. Post to the Cardano Forum CIP category to open a community discussion; update the Discussions field in the preamble with the resulting URL
2. Submit a pull request to the CIP repository once a forum discussion is underway
3. A reference Plutus validator implementing Tier 1 semantics is published and reviewed by the community
4. At least one Cardano wallet ships a production Tier 1 implementation
5. No unresolved security objections remain open in the CIP discussion thread

Tier 2 and Tier 3 adoption are left to the ecosystem and may be addressed in subsequent CIPs or revisions once Tier 1 is established.

---

## Open Questions / Future Work

- Standardized wallet UX guidelines for surfacing pending, declined, replaced, and amended transactions to both parties
- Payment request format: a standard URI or QR schema encoding recipient address, expected amount, and capability token public keys, enabling one-scan transaction initiation
- Standardized error codes for DECLINE and automated system rejection responses, enabling programmatic handling by merchant systems
- Privacy considerations: pending transactions are publicly observable on-chain by design; applications with confidentiality requirements should evaluate whether this is acceptable or whether off-chain coordination channels are preferable
- Cross-chain pending transactions (future scope; significant additional complexity)
- **MUTUAL co-signature accumulation and partial-consent retractability:** When `commit_authority`, `abort_authority`, or `decline_authority` is set to `MUTUAL`, the protocol requires all designated parties to co-sign the operation. The mechanism for accumulating partial signatures toward a completed MUTUAL operation is chain-specific; on UTXO chains this maps naturally to multi-sig script patterns, while on account-based chains the flow is less standardized. This specification does not define the intermediate state between the first and final co-signature; chain-specific implementation guidance is required. A related open question is whether a party who has already contributed their partial co-signature toward a MUTUAL operation may retract it before the remaining parties have signed and the operation completes. Two positions are defensible: irretractable partial consent (your signature is a permanent statement of intent; prevents sign/revoke harassment) vs. retractable partial consent (circumstances change; prevents a party from being indefinitely on record as wanting an operation that will never complete because others refuse). Since the operation has not yet executed in either case (the transaction state is unchanged until all signatures are collected) neither position introduces a safety risk, making this primarily a UX and social-layer question. Chain-specific implementations may choose either behavior; a future revision to this specification may standardize one approach once implementation experience accumulates.
- **Bilateral-consent decline (`decline_authority = MUTUAL` with sender inclusion):** The current specification excludes the sender from all `decline_authority` configurations, including `MUTUAL`, primarily for semantic cleanliness: the purpose of DECLINED is to unambiguously record that the counterparty refused the transaction. If the sender's co-signature were required to produce a DECLINED state, then DECLINED would mean "both parties agreed to call it a decline" rather than "the counterparty refused", which is a meaningfully different signal for observers, wallets, and downstream systems. With sender inclusion, a sender who withholds co-signature simply leaves their own funds in PENDING, which is an inconvenience for the recipient (no clean on-chain refusal record) but causes no financial harm to either party. There is nonetheless a legitimate use case for a *bilateral-consent-to-decline* model in symmetric high-value agreements (real estate, large contracts, formal settlements) where both parties wish to produce a mutually-attested termination record. A future revision could address this by introducing a named variant (e.g. `MUTUAL_WITH_SENDER`) for decline only, or introducing a new termination type beyond ABORTED & DECLINED such as MUTUAL_TERMINATION, or by recommending that parties needing bilateral termination records use `decline_authority = THIRD_PARTY_ONLY` with a mutually trusted arbitrator who acts only upon joint instruction from both principals; achieving the same outcome at the social layer without compromising the semantic clarity of the DECLINED state.

---

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
