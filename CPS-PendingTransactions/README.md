---
CPS: ?
Title: Irreversible Cryptocurrency Transactions Create Unrecoverable User Errors and Adoption Barriers
Category: Wallets
Status: Open
Authors:
  - Yurelle Gamier <yurelle.gamier@gmail.com>
Proposed Solutions:
  - CIP-?: Pending Transactions - Two-Phase Transaction Protocol (2PT)
Discussions:
  - https://forum.cardano.org/t/cip-proposal-pending-transactions-a-two-phase-transaction-protocol-for-safer-crypto-sends/153742
Created: 2026-03-28
License: CC-BY-4.0
---

## Abstract

Cryptocurrency transactions are one-shot and irreversible by design. Once a sender broadcasts a transaction and it is confirmed on-chain, the funds are permanently transferred with no undo, no grace period, and no recourse. This creates a class of severe, unrecoverable user errors (ex: funds sent to a mistyped address, a stale address, or the wrong network, are simply gone). Beyond individual errors, this all-or-nothing model is one of the most significant psychological barriers to mainstream cryptocurrency adoption. Traditional finance has long since addressed this with pending transactions, which allow a period of validation & verification of the transaction before funds are settled. No such mechanism exists as a standardized, first-class feature in any major cryptocurrency ecosystem.

## Problem

### The irreversibility gap

Current cryptocurrency transaction models, including Cardano's, finalize transfers immediately and permanently upon on-chain confirmation. There is no protocol-level mechanism for a sender to reserve funds for a recipient, allow the recipient to verify the transaction is correctly configured, and then commit (or abort) based on that verification.

This creates several distinct but related problems:

**Unrecoverable address errors.** Wallet addresses are long, opaque alphanumeric strings. Mistyped characters, stale addresses, an address for the wrong network, or addresses belonging to wallets the recipient no longer controls, result in permanent, total loss of funds. The only protection currently available is manual visual inspection by the sender alone; an error-prone, single-point-of-failure check.

**High psychological barrier to entry.** The all-or-nothing nature of sending cryptocurrency asks every user (from a seasoned developer to someone making their first transaction) to be perfectly correct with no safety net. This is categorically unlike any other payment mechanism most users have experience with. Traditional banking, card payments, and even cash, all provide either reversibility, physical confirmation, or bilateral acknowledgment. The absence of any equivalent in cryptocurrency is a serious adoption barrier.

**No proof-of-funds primitive.** In commercial and contractual contexts (ex: real estate, large purchases, service agreements) parties routinely need to demonstrate that funds are available and committed to a specific transaction. Current cryptocurrency offers no standardized way to do this. A wallet balance can be spent at any moment; there is no mechanism to lock funds against a specific pending obligation in a way that is cryptographically verifiable and observable by the counterparty.

**Unreliable automated payment flows.** Businesses accepting cryptocurrency must either trust that incoming payments are correctly configured (wrong amount, wrong address, and wrong network errors are common) or build bespoke verification logic with no standard interface. There is no protocol-level mechanism for a merchant system to verify a payment intent before it is irrevocably committed.

### Why existing mechanisms do not solve this

Hash Time-Locked Contracts (HTLCs), used extensively in the Lightning Network, provide conditional atomicity across multi-hop payment routes but are not designed for human address verification or bilateral handshaking. They do not expose a recipient-visible pending state in standard wallet UX.

Smart contract escrow patterns exist on EVM chains but are bespoke, non-standardized, and not surfaced as first-class wallet UX. The gap is not technical feasibility but the absence of a standard.

Payment channels require upfront channel establishment and are optimized for high-frequency bilateral payments, not single-transaction safety.

None of these mechanisms address the core problem: a standardized, wallet-level, protocol-native pending transaction mechanic that is familiar to lay users and usable without specialist knowledge.

## Use Cases

### Bilateral address verification

A sender wants to pay a recipient they have not paid before. Under the current model, the only way to verify the address is correct is for the sender to visually inspect a really long alphanumeric string.

A pending transaction allows the recipient to observe the incoming transaction against their own address and confirm it before the sender commits. The verification is bilateral and cryptographically grounded rather than unilateral and error-prone.

### Lay user sending funds for the first time

A user wants to send ADA to a friend. They have never sent cryptocurrency before. Under the current model, they paste a wallet address, enter an amount, and press send; at which point the transaction is final. If the address was wrong, the funds are gone forever.

A pending transaction mechanism would allow the user to issue a prepare transaction. The recipient's wallet shows the incoming amount as pending. The recipient confirms the address and amount are correct and signals back. The sender confirms, and the transfer completes. The experience mirrors a bank transfer with a pending, verification stage (familiar and safe).

### Proof of funds for a large purchase

A buyer wants to demonstrate to a seller that they have sufficient funds reserved for a high-value transaction. Under the current model, the seller can only see the buyer's wallet balance; which can be spent at any moment.

A pending transaction allows the buyer to lock funds against a specific transaction. The seller can observe on-chain that the funds are locked, cannot be double-spent, and are designated for this transaction. This provides stronger assurance than any existing cryptocurrency mechanism.

### Automated merchant payment verification

An e-commerce platform wants to accept ADA payments. Under the current model, the merchant cannot verify that an incoming payment is correct before it is irrevocably committed.

A pending transaction mechanism allows the merchant's system to detect a prepare transaction, verify the destination address, amount, and network, and automatically confirm or reject the transaction, before funds move; enabling a payment flow comparable to traditional payment processors.

## Goals

Goals are listed in order of importance.

1. **Reversibility before commitment.** A sender must be able to reserve funds and cancel the reservation without any loss, provided the transaction has not yet been confirmed by a second explicit action.

2. **Recipient-visible pending state.** A recipient must be able to observe an incoming pending transaction (including the amount and source) before the sender confirms it. This is the mechanism that enables bilateral verification.

3. **No change to existing transaction semantics.** Standard one-phase transactions must be unaffected. The mechanism must be strictly additive; existing wallets and tooling must continue to function without modification.

4. **Trustless and decentralized.** The pending state must be verifiable on-chain by any party without requiring trust in a third party. The mechanism must not introduce centralized intermediaries.

5. **Lay-user accessible.** The mechanism must be expressible in wallet UX terms that are familiar to users of traditional banking. The underlying protocol complexity should not be visible to end users.

6. **Composable with automation.** The mechanism should support automated confirmation by recipient systems (merchants, payment processors) without requiring manual intervention from the sender in the happy path.

**Non-goals:**

- Replacing or deprecating existing one-phase transaction flows
- Requiring all transactions to use the two-phase mechanism
- Providing reversibility after the sender has explicitly confirmed the transaction
- Defining wallet UX guidelines (out of scope for the protocol specification)

## Open Questions

1. **Minimum viable on-chain representation on Cardano.** What is the simplest Plutus validator that correctly implements Tier 1 semantics (prepare / confirm / abort with timeout), and can it be deployed as a shared reference script to minimize transaction fees for users?

2. **Wallet discovery and UX standardization.** How should wallets surface pending inbound transactions? What metadata standards are needed for wallet interoperability (i.e., for a prepare issued by Wallet A to be correctly displayed by Wallet B)?

3. **Timeout defaults and recommendations.** Should the ecosystem converge on recommended default timeout values for common use cases (retail payment, escrow, proof-of-funds)? What are the implications of indefinite-timeout transactions for chain state?

4. **Fee implications.** A two-phase transaction requires at least two on-chain operations instead of one. What are the realistic fee costs for Tier 1 transactions on Cardano, and are they acceptable for the retail use case?

5. **Privacy.** Pending transactions are publicly observable on-chain by design. For use cases requiring confidentiality (e.g. business-to-business payments where amount and counterparty should not be public), is an off-chain coordination layer needed, and how should it interact with the on-chain pending state?

6. **Cross-wallet authority delegation.** For Tier 2 capability tokens (ex: delegated commit authority), what out-of-band channel and data format should be standardized for sharing public keys between sender and recipient systems prior to PREPARE?

---

## Copyright

This CPS is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
