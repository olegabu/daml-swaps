# Canton / DAML Architecture Notes

Reference notes for whoever is building on this
project, so contract and client code stays consistent with how Canton
actually works rather than assumptions carried over from other DLT stacks
(Ethereum, Hyperledger Fabric).

## DAML authorization vocabulary

- **Signatory** — must consent for a contract to exist; bound by whatever
  the contract lets happen to them afterward.
- **Observer** — can see the contract; no rights over it.
- **Controller** — can exercise a specific choice; must be at least an
  observer, need not be a signatory. This is how delegation works.
- **Choice** — the only way state changes. `consuming` (default) archives
  the contract; `nonconsuming` leaves it live (for repeatable actions like
  periodic settlement or margin calls).
- Authorization is **non-transitive**: a signatory only pre-authorizes the
  *direct* consequences of a choice on their contract, not arbitrary
  downstream effects.
- **Propose-accept pattern**: the standard way to get a bilateral contract
  onto the ledger without one party unilaterally binding the other, and —
  more fundamentally — the only way to combine two parties' authority when
  they're on different participant nodes. See "Propose-accept: carrying
  authority across transactions" below.

## Authority vs. visibility (two separate things)

A lesson this codebase validated concretely, worth stating plainly because
they're easy to conflate:

- **Authority** = who consents to a transaction. When a choice is
  exercised, the resulting subtree is authorized by (the contract's
  signatories) ∪ (the choice's controllers). Because `IRSTrade` is signed
  by *both* `fixedRatePayer` and `floatingRatePayer`, any choice on it —
  even one controlled by a third party like the oracle — already carries
  both counterparties' authority. That's why the oracle can move *their*
  cash: exercising `AdjustBalance` (controller = the account owner, i.e.
  `fixedRatePayer` or `floatingRatePayer`) is authorized without either
  party submitting anything.
- **Visibility** = who can *see* a contract to act on it. Independent of
  authority. The submitting/reading parties must be stakeholders
  (signatory or observer) of any contract the transaction fetches or
  exercises. The oracle has full authority to move the cash but, unless
  it's also a stakeholder of the `CashHolding`, it literally can't see it
  to do so — the exercise fails `CONTRACT_NOT_FOUND` / "not visible to the
  reading parties".

Two ways to grant the needed visibility: (a) the submitter reads-as the
owners (`readAs` claims in the JWT / `submitMulti`), or (b) make the third
party a declared **observer** on the asset. This project uses (b) — the
oracle is an observer on both cash accounts — because it's explicit in the
data model and works from Navigator (which submits with `actAs` = only the
logged-in party, no `readAs`). The trade-off: the oracle can then see both
counterparties' balances, which for a settlement/calc agent is realistic.

## Propose-accept: carrying authority across transactions

A **single Daml transaction is submitted by one participant node**, which
can only supply `actAs` authority for parties *it hosts* (and that the
caller's token covers). A choice with two independent controllers on two
different parties — e.g. this codebase's original

```haskell
choice Novate : ContractId IRSTrade
  with outgoingParty : Party; incomingParty : Party; incomingCashCid : ContractId CashHolding
  controller outgoingParty, incomingParty
```

— can therefore only ever be submitted directly when **both** parties are
hosted on the same participant, because only then can one submission
`actAs` both of them at once. `Daml Script`'s `submitMulti [bob, charlie] []
…` (and this project's earlier `daml repl` novation demo) works *only*
because the sandbox hosts every party on a single node — it is a testing
convenience, not something a real cross-organization deployment can do.
The moment Bob and Charlie sit on distinct participants (see "Sync domains
/ Global Synchronizer" and `NEXT_STEPS.md` item 2), Bob's node cannot
`actAs` Charlie, and a direct multi-controller choice like that becomes
unsubmittable — full stop, not just inconvenient.

The fix is to split one logically-joint action into **two sequential
single-party transactions**, using a contract as the carrier that freezes
the first party's authority so the second transaction can pick it up. When
a later choice is exercised on that contract, the transaction's available
authority = (the contract's **signatories**) ∪ (the choice's controllers)
— so a party who is merely a *signatory* of the carrier contract
contributes its authority to whatever that contract's controller-only
choices do next, with no further submission from that party required.
Nothing restricts this to two hops, either — chain it again and a third
party's consent joins the same way. Three instances of this pattern in
this codebase:

- **Trade execution** — `TradeProposal` (signatory = `proposer` alone) /
  `AcceptTrade` (controller = `counterparty`). Accepting creates `IRSTrade`
  signed by `{proposer, counterparty}`; available authority at that point =
  `{proposer}` (from the proposal's signatory) ∪ `{counterparty}` (the
  accept choice's controller) = exactly the two signatories needed. Neither
  party ever needed to `actAs` the other.

- **Novation — tri-party, chained twice.** Real ISDA novations need three
  consents, not two: the **transferor** (stepping out), the **transferee**
  (stepping in), and the **remaining party** — whose counterparty credit
  risk changes as a result, so they get a genuine say, not just inherited
  authority. That's `ProposeNovation` (a *consuming* choice on the live
  `IRSTrade`, `controller transferor`) → `NovationProposal` →
  `AcceptNovation` or `RejectNovation` (`controller transferee`) →
  `NovationConfirmation` → `ConfirmNovation` or `DeclineNovation`
  (`controller remainingParty`). `transferor`/`transferee` are ISDA's own
  Novation Agreement terms for the two roles. Three details make this one
  work correctly:

  1. **Archiving the old trade needs no new authority.** Because
     `ProposeNovation` is exercised *on* `IRSTrade`, whose signatories are
     already `{fixedRatePayer, floatingRatePayer}`, the archive is
     authorized regardless of which single party is the choice's controller
     (see "Authority vs. visibility" above) — `transferor` alone is
     enough to trigger it. This also fixes `remainingParty` for the whole
     flow: it's computed once here (whichever of `fixedRatePayer`/
     `floatingRatePayer` isn't `transferor`) and stored on `NovationProposal`,
     the same way `TerminationProposal.otherParty` is.
  2. **`AcceptNovation` can't create the final trade directly** — that would
     silently skip the remaining party's consent, the exact gap a tri-party
     flow exists to close. Instead it creates `NovationConfirmation`,
     signed by all **three** parties at once: `{fixedRatePayer,
     floatingRatePayer}` (= `{transferor, remainingParty}`, carried forward
     from `NovationProposal`'s signatories) plus `transferee`, `AcceptNovation`'s
     own controller. That three-way signature is available for free the
     moment `AcceptNovation` runs — the same trick as step 1, one hop later.
  3. **Both proposal-stage AND confirmation-stage rejection recreate the
     ORIGINAL trade, not just archive things.** `NovationProposal`'s
     signatories carry `transferor`'s authority forward for `RejectNovation`
     (transferee declines outright); `NovationConfirmation`'s signatories
     carry it forward *again* for `DeclineNovation` (remaining party vetoes
     even after the transferee already accepted) — in both cases recreating
     `IRSTrade` signed by `{transferor, remainingParty}` without transferor
     ever submitting a second (or third) time. `ConfirmNovation` needs only
     `remainingParty`'s own authority plus `transferee`'s, carried forward
     from `NovationConfirmation`'s signatories, to create the novated trade
     signed by `{remainingParty, transferee}`.

  The result: `Novate`'s old single dual-controller choice becomes five
  single-controller choices across three transactions, each submittable by
  exactly one party from its own participant — no co-hosting required,
  genuinely three-way consent, not two-way-plus-inherited-authority.

- **Termination** — identical shape to the novation *proposal* stage, one
  level simpler (no third party to chase):
  `ProposeTermination` (`controller proposer`, consuming on `IRSTrade`) /
  `AcceptTermination` or `RejectTermination` (both `controller
  otherParty`) on the `TerminationProposal` it creates. `otherParty` is
  computed once at proposal time (whichever of `fixedRatePayer` /
  `floatingRatePayer` isn't `proposer`) and stored, same role
  `transferee` plays for novation's first hop — `AcceptTermination` has
  nothing left to do but archive the proposal (`pure ()`, no successor),
  while `RejectTermination` recreates
  the original trade exactly like `RejectNovation` does, for the same
  reason: `TerminationProposal` carries both original signatories forward,
  so the proposer never needs to submit again either way.

  Both replacements retire this codebase's last two direct multi-controller
  choices (the old `Novate` and `Terminate`) — every lifecycle choice is
  now submittable by a single party from its own participant, no
  co-hosting required anywhere in the trade's life.

For a genuinely **atomic** cross-participant multi-party transaction
(rather than two sequential ones), Canton 3.x's external-party /
interactive-submission API lets each required party sign a prepared
transaction hash with its own key before one combined submission — this
project targets SDK 2.10.4, where propose-accept is the available pattern.

## Oracle / calculation-agent pattern

`SettleCashflow` and `SettleMargin` are controlled *solely* by an oracle
party that submits only an objective external number (a rate fixing, a
mark-to-market) and never a party or contract ID — so it stays oblivious to
counterparty identities. The contract itself derives who owes whom and by
how much. This mirrors real practice: rate fixings come from an independent
administrator, and for centrally-cleared trades the CCP computes and calls
margin unilaterally. (For *uncleared* bilateral trades, margin is instead
each party calculating independently and reconciling disputes — a
deliberate simplification here.) Guarding these inputs against fat-fingers
(e.g. a rate must be a decimal fraction in [0,1]) keeps a typo from
silently defaulting the trade.

## Settlement / cash leg

DAML contracts don't move real money by themselves. Real settlement =
transferring an on-ledger cash **token** (tokenized bank deposit, or a
stablecoin like USDC/USDCx) atomically, in the same transaction as the
contract's state update. Tokenized deposits carry depositor-level legal
protection (bank liability, deposit insurance); stablecoins make the
holder a creditor of a private issuer — a materially weaker claim, worth
being deliberate about which one this project's `CashHolding` is meant to
stand in for.

This project implements that atomic DvP: `SettleCashflow` / `SettleMargin`
compute the amount, then debit the payer's `CashHolding` and credit the
receiver's (via `AdjustBalance`) and recreate the trade — all in one
transaction. If the payer's account can't cover the amount, the same
transaction instead archives the trade into a terminal `SettlementFailure`
(no successor `IRSTrade`, so the ledger structurally prevents any further
lifecycle action). Each party keeps a single account; the trade stores
both accounts' contract IDs and refreshes them on every settle.

## Core concepts: 

### No global ledger

There is no single shared database. State is the **Active Contract Set
(ACS)** — the set of live (created, not-archived) contract instances — but
**each participant node maintains only its own local ACS**, containing
only contracts where its own hosted parties are signatories, observers, or
controllers. Every state change is archive-old + create-new; DAML has no
in-place mutation.

### Transaction flow (what happens when a choice is exercised)

1. Submitting participant builds the transaction as a **tree** of action
   nodes (Create/Exercise/Fetch), decomposes it into **views** along
   informee boundaries (who's a stakeholder of which node), and encrypts
   each view separately.
2. Non-informees don't get nothing — they get a **Merkle-tree hash
   commitment** to the parts they can't see, enough to verify structural
   consistency without learning content.
3. The **sequencer** routes each encrypted view only to the participants
   addressed for it, in a fair, timestamped order. It never sees plaintext
   and never validates business logic — purely ordering + delivery,
   addressed using topology-registered participant routing info.
4. Each recipient participant locally validates its own view (DAML
   authorization + `ensure`/`assertMsg` checks) and sends a signed
   **confirmation** (or rejection) to the **mediator**.
5. The **mediator** checks that every required confirmation (every
   signatory and controller the transaction needs — not a majority
   threshold, all-or-nothing) arrived within the time window, and issues a
   commit/abort **verdict**, distributed back via the sequencer.
6. On commit, each affected participant updates its own local ACS. Clients
   see this via the Ledger API's active-contract/transaction event stream.

Mediator threat model: it's a real **liveness/censorship** risk (can
delay/withhold verdicts) but not a **safety** risk in the naive sense — a
malicious mediator can't forge a valid commit, because verdicts are built
from cryptographically signed, independently-checkable confirmations. The
mediator role itself can run as a small BFT cluster rather than one node.

### Sync domains / Global Synchronizer

A **sync domain** = sequencer + mediator (+ topology manager), the
coordination layer participants connect to. Domains can be private
(single-operator) or the public **Global Synchronizer** (BFT consensus run
by a Super Validator consortium, MainNet currently invite-only via
sponsorship). Two participants can only transact if connected to a common
domain. Community edition Canton bundles sequencer+mediator+topology
manager into one **embedded domain** node; separating them is an
Enterprise feature.

### Identity & cryptography

- **Client app → own participant**: JWT bearer tokens (OAuth2/OIDC),
  `actAs`/`readAs` claims scoped to parties. The participant validates,
  never issues, tokens — issuance is delegated to an external IdP.
  Local dev: unsafe HS256 shared-secret tokens (fast) or a self-hosted
  Keycloak instance (closer to production OIDC).
- **Node ↔ node**: mutual TLS at the transport layer, plus every protocol
  message (confirmations, topology transactions) individually signed by
  the sender's registered key. Node identity = hash of its root signing
  key ("namespace"). Sequencer authenticates connecting nodes via
  challenge-response signature verification against topology state.
- **Party identity**: a Party ID (e.g. `Alice::1220a3f9...`) is a name +
  key fingerprint, bound to signing/encryption keys via signed topology
  transactions (`PartyToKeyMapping`, `OwnerToKeyMapping`). Cryptography
  proves key possession / non-repudiation — it does NOT replace KYC; the
  binding of a Party ID to a real legal entity is an off-ledger trust
  decision made at onboarding.
- **Participant vs. party**: one participant node (usually one per
  organization) can host many parties. Routing (`PartyToParticipant`) is a
  signed topology mapping the sequencer looks up — not a decision it makes
  itself.

### Client-side interaction

- **Ledger API**: gRPC or JSON-over-HTTP. Two core commands:
  `CreateCommand`, `ExerciseCommand`. `daml codegen` generates typed
  bindings (TS/JS, Java) from the DAML source.
- **Async by design**: submit a command, get a completion later via a
  streamed completion event, correlated by a client-set command ID.
  Handle timeout-as-uncertain-outcome (a timeout doesn't guarantee nothing
  happened) — use command deduplication IDs, don't blindly retry.
- **Streaming**: WebSocket endpoints for active-contract/transaction
  streams and command completions (JSON API). Query first for current
  state on startup, then stream for changes.
- **Navigator** still ships and works in SDK 2.10.4 (this project uses it
  for the demo — log in as `oracle` to drive settlements), but it prints a
  deprecation warning and is removed in Daml 3.0. For anything beyond a
  local demo, use Postman/Insomnia or `curl` against the JSON API,
  `daml script` for repeatable typed interactions, or a small custom
  `@daml/react` app. Note Navigator submits with `actAs` = only the
  logged-in party (no `readAs`), which is why the oracle needs *observer*
  visibility on the cash accounts rather than read-delegation (see
  "Authority vs. visibility"). **DAML Triggers are also now deprecated**
  and were never suited to automation needing external data anyway —
  scheduled off-ledger services (plain client code + cron) are the right
  pattern for things like daily margin calculation, which needs an
  external pricing engine feeding the oracle.
- **DAML Shell** is a read-only query tool over a Participant Query Store
  (indexed DB) — for inspecting already-committed data, not for
  submitting new commands.

### Deployment

1. `daml build` → `.dar` file (DAML-LF bytecode).
2. Upload the DAR to every participant that needs it (`participant.dars.upload`
   or Ledger API package management) — **every counterparty's participant
   needs a compatible DAR**, not just your own; this is a coordinated
   multi-party rollout, not a single-org deploy.
3. Vet the package **per sync domain** the participant is connected to.
4. Packages are immutable once deployed — upgrades get new package IDs;
   Smart Contract Upgrade (SCU) handles compatible changes, incompatible
   ones need explicit migration (archive-old/create-new) coordinated
   across every hosting participant.
